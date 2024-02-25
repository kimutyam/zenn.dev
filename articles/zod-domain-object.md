---
title: "Zodでロバストなドメインオブジェクトを定義する"
emoji: "🐳"
type: "tech"
topics: ["TypeScript", "Zod", "NeverThrow", "DDD"]
published: true
published_at: 2024-02-26 10:00
---

# 課題意識

特定の商品を数量を指定して購入できるECサービスのドメインモデルを実装するとします。TypeScriptで構築する際に、「数量」を単にnumber型で扱うことは可能ですが、よりロバストな設計を目指した結果、以下の2つの方法論に行き着きました。

1. Refinements(値の制約を表す): 「数量」は一般的に自然数です。1度の注文で指定できる上限を設けるビジネスルールがあると仮定します。この場合、number型に「自然数」「上限付き」の制約を加えた値として表現したいです。

2. Branded Types: (同じ構造の型を区別する): 「価格」などの他のnumber型と混同されないように、これらの数値を型レベルで区別したいです。JavaやC#に見られる公称型の概念をTypeScriptで模倣するBranded Typesのテクニックを用いることで、これらの誤った利用を型システムで防ぐことができます。

このように、システムを利用するコンテキストに応じて限定的な制約をもたせることで、システムの安全性と信頼性を向上させることができます。そして、これらの設計上の課題は[Zod](https://zod.dev/)で簡単に解決できます。

:::message
この記事のサンプルコードは次のライブラリのバージョンを前提としています。

- Zod: 3.22.4
- Jest: 29.7.0
- neverthrow: 6.1.0
- remeda: 1.26.0
:::

# 導入
## Zodとは

[Zod](https://zod.dev/)はTypeScriptファーストで設計されたスキーマ宣言型のバリデーションライブラリです。このライブラリでは、型情報を持たないデータを事前に定義したスキーマで検証し、TypeScriptの型情報を持つデータへと安全に変換することができます。APIリクエスト・レスポンス、データベースからのデータ取得、設定ファイルなど、TypeScriptの型付けコンテキスト外でのデータのバリデーションで使われることが多いです。

Zodではスキーマを最初に定義、そのスキーマから型を推論するようなアプローチを取ることが一般的です。型推論は[`inter`](https://zod.dev/?id=type-inference) メソッドを利用します。

```typescript
import { z } from 'zod';

export const productSchema = z.object({
  id: z.string(),
  price: z.number(),
});

export const orderItemSchema = z.object({
  product: productSchema,
  quantity: z.number(),
});

// type Product = {
//   id: string, 
//   price: number
// }
export type Product = z.infer<typeof productSchema>

// type OrderType = {
//   product: {id: string, price: number},
//   quantity: number
// }
export type OrderItem = z.infer<typeof orderItemSchema>;
```

スキーマの型はZodTypeという抽象型で表現されます。その型に内包されている [`parse`](https://zod.dev/?id=parse) メソッドを呼び出して入力値のデータが有効であることを確認します。有効であれば、スキーマの型情報に基づく型の値が返されます。そうでなければ、バリデーションの問題についての詳細な情報を含む `ZodError` インスタンスがthrowされます。では、挙動を単体テストで確認します。

```typescript
describe('parse', () => {
  it('成功', () => {
    const orderItem = orderItemSchema.parse({
      product: {
        id: 'ABC',
        price: 1000,
      },
      quantity: 10,
    });
    expect(orderItem).toStrictEqual({
      product: {
        id: 'ABC',
        price: 1000,
      },
      quantity: 10,
    });
  });

  it('エラー: quantityが文字列', () => {
    expect(() =>
      orderItemSchema.parse({
        product: {
          id: 'ABC',
          price: 1000,
        },
        quantity: '1個',
      }),
    ).toThrow(z.ZodError);
  });
});
```

[`safeParse`](https://zod.dev/?id=safeparse) メソッドを利用すると、パースエラー時ではthrowしません。戻り値は[判別可能な共用型](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#discriminated-unions)として、パースに成功したスキーマの型情報に基づく型の値か、`ZodError` インスタンスのどちらかになります。

```typescript
describe('safeParse', () => {
  it('成功', () => {
    const result = orderItemSchema.safeParse({
      product: {
        id: 'ABC',
        price: 1000,
      },
      quantity: 10,
    });
    assert(result.success);
    expect(result.data).toStrictEqual({
      product: {
        id: 'ABC',
        price: 1000,
      },
      quantity: 10,
    });
  });

  it('エラー: priceとquantityが文字列', () => {
    const result = orderItemSchema.safeParse({
      product: {
        id: 'ABC',
        price: 'Priceless',
      },
      quantity: '1個',
    });
    assert(!result.success);
    const formattedError = result.error.format();
    expect(formattedError).toEqual(
      expect.objectContaining({
        _errors: [],
        product: expect.objectContaining({
          price: expect.objectContaining({
            _errors: expect.arrayContaining(['Expected number, received string']),
          }),
        }),
        quantity: expect.objectContaining({
          _errors: expect.arrayContaining(['Expected number, received string']),
        }),
      }),
    );
  });
});
```

上記のテストで、[`assert`](https://nodejs.org/api/assert.html) メソッドでresult.successを検査しているのは、定義通り成否を検査するためだけでなく、判別可能な共用型の絞り込みをするためでもあります。

## Refinements

RefinementsはRefinement TypesのTypesを抜いた概念として取り上げています。  
Refiement Typesは値の制約を型で表現する方法です。冒頭の課題意識で紹介した「数量」に関する「自然数」「上限付き」といった制約を型レベルで表現することで、コンパイラが検査できるようになります。他言語では、Nikita Volkov氏によるRefinement TypesのHaskellライブラリである[refined](https://github.com/nikita-volkov/refined)や、それの移植が動機で開発されたScala版の[refined](https://github.com/fthomas/refined)のような高いシェアのライブラリが存在しますが、TypeScriptでそれらしきライブラリの存在を認知できていません。また、[refinement types #7599](https://github.com/microsoft/TypeScript/issues/7599)でTypeScriptでのRefinement Typesが提案されていますが、現時点では動きが見られません。

Zodの[refine](https://zod.dev/?id=refine)メソッドのリファレンスではRefinement TypesはTypeScriptの型システムでは実現できないとしたうえで、Zodスキーマにバリデーションチェックを定義してRefinementsを実現する旨が説明されています。

> Zod lets you provide custom validation logic via refinements. (For advanced features like creating multiple issues and customizing error codes, see .superRefine.)
> 
> Zod was designed to mirror TypeScript as closely as possible. But there are many so-called "refinement types" you may wish to check for that can't be represented in TypeScript's type system. For instance: checking that a number is an integer or that a string is a valid email address.
> 
> For example, you can define a custom validation check on any Zod schema with .refine

説明のための前置きは終え、ZodでRefinementsを実現する方法について見ていきます。入力値のバリデーションルールを定義するために利用していた方にとっては馴染み深い一般的な方法かもしれませんが、ZodではスキーマにRefinementsを定義します。  
次のような制約を装飾するために改修します:

- 商品ID: UUIDであること
- 商品価格: 1000円から10万円の範囲内の整数
- 数量: 1~10の範囲内の整数 

```typescript
import { z } from 'zod';

export const productSchema = z.object({
  id: z.string().uuid(),
  price: z.number().int().min(1000).max(100_000),
});

export const orderItemSchema = z.object({
  product: productSchema,
  quantity: z.number().int().min(1).max(10),
});
```

この時、`z.infer<typeof productSchema>` と `z.infer<typeof orderItemSchema>` で得られる型情報はスキーマに制約を装飾する前と変化はありませんが、`parse` や `safeParse` メソッドで検査する項目に含まれます。制約違反の時の挙動を示すためにエラーのアサーションのしやすい `safeParse` メソッドを使って、単体テストの例示をします。

```typescript
describe('safeParse', () => {
  it('全てのプロパティで制約エラー', () => {
    const result = orderItemSchema.safeParse({
      product: {
        id: 'ABC',
        price: 1,
      },
      quantity: 100,
    });
    assert(!result.success);
    const formattedError = result.error.format();
    expect(formattedError).toEqual(
      expect.objectContaining({
        _errors: [],
        product: expect.objectContaining({
          id: expect.objectContaining({
            _errors: expect.arrayContaining(['Invalid uuid']),
          }),
          price: expect.objectContaining({
            _errors: expect.arrayContaining(['Number must be greater than or equal to 1000']),
          }),
        }),
        quantity: expect.objectContaining({
          _errors: expect.arrayContaining(['Number must be less than or equal to 10']),
        }),
      }),
    );
  });
});

```


### エラーメッセージのカスタマイズ

上記で `expect` しているエラーメッセージはZodがデフォルトで出力する値ですが、制約違反時にエラーを適切に通知するために、エラーメッセージをは次のようにカスタマイズができます。

```typescript
export const productSchema = z.object({
  id: z.string().uuid('uuid形式にしてください'),
  price: z
    .number()
    .int('整数で指定ください')
    .min(1000, '1000円未満の商品を扱えません')
    .max(100_000, '10万円を超える商品を扱えません'),
});
```

### Refinementsのカスタマイズ

`uuid` や、`int`、`min`、`max` はZodがデフォルトで提供しているメソッドですが、先程引用のために取り上げた[`refine`](https://zod.dev/?id=refine) メソッドを使って独自の制約を定義することも可能です。

```typescript
const sortedStringSchema = z.string().refine(
  (arg) => [...arg].sort().join('') === arg,
  (arg) => ({ message: `ソートされていません: ${arg}` }),
);
```

ここでは紹介しませんが、[`superRefine`](https://zod.dev/?id=superrefine) メソッドを使うとより高度な設定を記述できます。


## Branded Types

TypeScriptの型システムは構造的部分型が採用されていて、型の互換性は型の構造に基づいて判定されます。一方、JavaやC#の型システムに見られる公称型は型の名前に基づいて区別され、構造に同じであろうが型名が異なれば互換性がないとみなされます。冒頭の課題意識で説明したように誤用を防ぎたい場面で、型を区別するためのBranded Typesのテクニックは役に立ちます。
Branded Typesを導入する方法やメカニズムについては次の記事で丁寧に語られていて参考になるので、気になる方は確認してください。:

- [公称型をTypeScriptで実現するための基礎](https://qiita.com/suin/items/ae9ed911ebab48c98835)
- [TypeScript の Type Branding をより便利に活用する方法のまとめ](https://zenn.dev/noshiro_piko/articles/typescript-branded-type-int)

さて、Zodではスキーマに対して、[`brand`](https://zod.dev/?id=brand) メソッドを宣言するだけで、Branded Typesを手に入れられます。`brand` メソッドの内部では、推論された型に交差型で「ブランド」を付加することで機能させています。このようにして、ブランド化されていないデータ構造をスキーマの推論された型に割り当てることができなくしています。

```typescript
import { z } from 'zod';

export declare const ProductIdBrand: unique symbol;
export declare const PriceBrand: unique symbol;

export declare const OrderQuantityBrand: unique symbol;

export const productSchema = z.object({
  id: z.string().uuid().brand(ProductIdBrand),
  price: z.number().int().min(1000).max(100_000).brand(PriceBrand),
});

export const orderItemSchema = z.object({
  product: productSchema,
  quantity: z.number().min(1).max(10).brand(OrderQuantityBrand),
});

// type OrderItem = {
//   product: {
//     id: string & z.BRAND<typeof ProductIdBrand>;
//     price: number & z.BRAND<typeof PriceBrand>;
//   };
//   quantity: number & z.BRAND<typeof OrderQuantityBrand>;
// };
export type OrderItem = z.infer<typeof orderItemSchema>;
```

なお、パースした成功したデータに対してブランドが付与されるため、`parse` メソッドや `safeParse` メソッドにおけるスキーマ検証ロジックには影響はしません。これはBranded Typesが欲しい時にいつでも追加ができることを意味します。

# 実践
## ドメインオブジェクトを実装する

RefinementsとBranded Typesのプラクティスを利用して値オブジェクト「数量」の定義をしてみます。値オブジェクトは、特定の型を用意し、その型内でビジネスルールをカプセル化することで、コードの整理とビジネスロジックの明確化する役割を果たします。

```typescript
import { z } from 'zod';

export declare const OrderQuantityBrand: unique symbol;

const schema = z.number().int().min(1).max(10).brand(OrderQuantityBrand);

export type OrderQuantity = z.infer<typeof schema>;

export type OrderQuantityInput = z.input<typeof schema>;

const build = (input: OrderQuantityInput): OrderQuantity => schema.parse(input);

const safeBuild = (input: OrderQuantityInput): z.SafeParseReturnType<OrderQuantityInput, OrderQuantity> =>
  schema.safeParse(input);

export const OrderQuantity = {
  build,
  safeBuild,
  schema,
} as const;
```


ここでは新たに、`.build()` と `.safeBuild()` のメソッドを定義しています:

- `.build()`: 不変条件違反の場合は例外が送出されます。確実に不変条件を満たす箇所で定義し、違反した場合はバグ(回復不能)として扱い、プログラムを停止します。
-  `.safeBuild()`: 不変条件違反の場合は、ZodErrorが戻ってきます。外部入力値等からオブジェクトを生成を試み、失敗した場合は要件に合わせてエラーハンドリングします。

`.build()` と `.safeBuild()`で引数に取る `OrderQuantityInput`はnumber型と評価されます。`.parse()` と `.safeParse()` はunknown型のデータを受け取るシグニチャになっているため、`.build` と `.safeBuild`でラップしています。

さて、`build()` と　`safeBuild()` を利用したドメインロジックを示すために、「商品」と「数量」をまとめた値オブジェクトの「注文項目」を次のように実装してみます。:

- buildSingle: 1個の商品で「注文項目」インスタンスを生成する
- add: 指定した数量を追加する
- calculateTotal: 合計金額を計算する

```typescript
const schema = z.object({
  quantity: OrderQuantity.schema,
  // NOTE: OrderQuantity.schemaと類似した定義になるため割愛
  product: Product.schema,
});

export type OrderItem = z.infer<typeof schema>;

const buildSingle = (product: Product): OrderItem => ({
  product,
  quantity: OrderQuantity.build(1)
})

const add =
  (quantity: OrderQuantity) =>
  (orderItem: OrderItem): OrderItem | z.ZodError<OrderQuantityInput> => {
    const result = OrderQuantity.safeBuild(orderItem.quantity + quantity);
    return result.success ? { product: orderItem.product, quantity: result.data } : result.error;
  };

const calculateTotal = ({ product, quantity }: OrderItem): number => product.price * quantity;

export const OrderItem = {
  buildSingle,
  add,
  total: calculateTotal,
  schema,
} as const;
```

`buildSingle` メソッドでは、`OrderQuantity.build` メソッドを利用しています。ここでは外のコンテキスト、つまり入力値等には関係なく、数量が1つであると決められたルールに基づいてファクトリを実装しています。「数量」の不変条件違反になった場合はバグとしてみなすため、`OrderQuantity.build` を利用します。  
`add` メソッドでは、`OrderQuantity.safeBuild` メソッドを利用しています。ここで `build` メソッドを使うか、`safeBuild` を使うかはビジネスルールやユースケースにも寄りますが、ここでは外部入力によって追加する数量を外部入力するユースケースを想定して、エラーハンドリングに開かれた戻り値にする設計の選択をしています。戻り値は、`OrderItem | z.ZodError<OrderQuantityInput>` になっていますが、必要に応じて、カスタムエラー型に変換する実装にしてもいいでしょう。このままだと、Union型の判別は `add` メソッドを使うクライアントに委ねることになりますが、判別可能な共用型でないとロバストに判別ができません。Zodの`safeParse`メソッドの戻り値である `SafeParseReturnType` 自体は判別可能なUnion型ですが、それを判別して関数を適用して再度 `SafeParseReturnType` 型に入れるような処理ができるメソッドがないため、無策に利用していると、例のようにUnion型で取り扱うことになります。これを解決するために次項でResult型を紹介します。

## パースエラーはResult型にWrapする

Result型は成否の判別可能な抽象型です。TypeScriptはResult型を標準でサポートしていないため、自前で用意するか、ライブラリを導入する必要があります。Result型を自前で用意するのでもいいのですが、次のような処理を抽象化した関数を用意するのは面倒です。

- Result型に値を入れる
- Result型に関数を適用する
- Result型の値を取り出す

ライブラリは筆者の観測範囲だと次の選択肢が有力な候補です。:

- [NeverThrow](https://github.com/supermacro/neverthrow)
- [fp-ts](https://github.com/gcanti/fp-ts)
- [option-t](https://github.com/option-t/option-t)

Result型をサポートするライブラリの比較やそれぞれの利用方法は他でも多く語られているため割愛します。今回はNeverThrowを利用してドメインロジックを組み立てます。(私はoption-tをプロダクションコードで利用したことがないため、fp-tsとNeverThrowを比較したうえで関数型の知識を多く求めないNeverThrowで説明します。)

先程定義した `OrderQuantity` の `safeBuild` メソッドの戻り値を、NeverThrowのResult型になるように変更をします。

```typescript
const safeBuild = (input: OrderQuantityInput): Result<OrderQuantity, z.ZodError<OrderQuantityInput>> =>
  buildFromZodDefault(schema.safeParse(input));
```

`buildFromZodDefault` は、Zodの `SafeParseReturnType` 型から、NeverThrowの `Result` に変換するユーティリティメソッドとして定義したものです。実際は次のように定義しています。

```typescript
import type { Result } from 'neverthrow';
import { err, ok } from 'neverthrow';
import { identity } from 'remeda';
import type { z } from 'zod';

export const buildFromZod = <Input, Output, E = z.ZodError<Input>>(
  result: z.SafeParseReturnType<Input, Output>,
  f: (e: z.ZodError<Input>) => E,
): Result<Output, E> => (result.success ? ok(result.data) : err(f(result.error)));

export const buildFromZodDefault = <Input, Output>(
  result: z.SafeParseReturnType<Input, Output>,
): Result<Output, z.ZodError<Input>> => buildFromZod(result, identity);
```

- buildFromZod: Zodの`SafeParseReturnType` が成功の場合はNeverThrowのOk型に、失敗の場合は、NeverThrowのErr型に変換します。Err型に含める具体的なエラーオブジェクトをZodErrorから変換するための関数fを引数に持ちます。
- buildFromZodDefault: Err型に含める具体的なエラーオブジェクトをZodErrorのままに戻します。他はbuildFromZodと同じです。実装の便宜上、Remedaの[identityメソッド](https://remedajs.com/docs/#identity)を使っています。

続いて、`OrderQuantity` の `add` メソッドの戻り値もResultになるように修正します。

```typescript
const add =
  (quantity: OrderQuantity) =>
  (orderItem: OrderItem): Result<OrderItem, z.ZodError<OrderQuantityInput>> =>
    OrderQuantity.safeBuild(orderItem.quantity + quantity).map((newQuantity) => ({
      ...orderItem,
      quantity: newQuantity,
    }));
```

このように変更することで、ドメインロジック内で制約違反になったエラーをResult型のコンテキストで統一し、クライアントでの判別及び処理を宣言的にすることができます。

## 入力値をドメインオブジェクトに変換する

先程定義した `OrderItem.schema` を利用してドメインオブジェクトを生成するサンプルをテストコードで示します。思い出すために、簡略化した`OrderItem.schema`のコードを再掲します。

```typescript
const schema = z.object({
  quantity: OrderQuantity.schema,
  // NOTE: OrderQuantity.schemaと類似した定義になるため割愛
  product: Product.schema,
});

/** 略 */

export const OrderItem = {
  /** 略 */
  schema,
} as const;
```

続いてテストコードです。

```typescript
it('注文項目をパースする', () => {
  const data: unknown = {
    product: {
      id: '8456C9A7-5135-4067-913A-378ED93A1DAC',
      price: 1_000,
    },
    quantity: 3,
  };

  const result = OrderItem.schema.safeParse(data);
  assert(result.success);

  const orderItem: OrderItem = result.data;

  expect(orderItem.product.id).toBe('8456C9A7-5135-4067-913A-378ED93A1DAC');
  expect(orderItem.product.price).toBe(1_000);
  expect(orderItem.quantity).toBe(3);
});
```

入力値のスキーマとドメインオブジェクトのスキーマに同じもしくは互換性がある場合は、パースに成功します。さて、入力値のデータ構造とドメインオブジェクトのデータ構造は異なる場合が往々にしてあります。次のような入力値のスキーマを ドメインオブジェクトの `OrderItem` 型に変換する処理を考えてみましょう。

```typescript
// 価格が存在しない
const schema = z.object({
  id: ProductId.schema,
  // 値オブジェクトの制約よりも強い入力制限をするユースケースを想定
  quantity: z.number().int().min(1).max(2),
});
```

この場合は、上記スキーマをDTO(Data Transfer Object)のそれと見立て、変換用の関数を用意します。

```typescript
export type OrderQuantityDto = z.infer<typeof schema>;

const toOrderItem =
  (price: Price) =>
  ({ id, quantity }: OrderQuantityDto): OrderItem => ({
    product: {
      id,
      price,
    },
    // 値オブジェクトの制約を満たすことが自明であるため、safeBuildでなくて、buildを利用
    quantity: OrderQuantity.build(quantity),
  });

export const OrderQuantityDto = {
  schema,
  toOrderItem,
} as const;
```

上記で定義した `schema` と `toOrderItem` を利用してドメインオブジェクトを生成するサンプルをテストコードで示します。

```typescript
it('構造の異なる入力値から注文項目を組み立てる', () => {
  const data: unknown = {
    id: '8456C9A7-5135-4067-913A-378ED93A1DAC',
    quantity: 1,
  };
  const result = OrderQuantityDto.schema.safeParse(data);
  assert(result.success);

  const orderItem: OrderItem = R.pipe(result.data, OrderQuantityDto.toOrderItem(Price.build(100)));
  expect(orderItem.product.id).toBe('8456C9A7-5135-4067-913A-378ED93A1DAC');
  expect(orderItem.product.price).toBe(1_00);
  expect(orderItem.quantity).toBe(1);
});
```

本編と関係ないですが、OrderItemに変換する処理フローを簡潔に記述するために、[Remedaのpipe関数](https://remedajs.com/docs/#pipe)を利用しています。


# 最後に

ドメインオブジェクトをロバストな設計を目指すための方法論としてRefinementsとBranded Typesを取り上げ、それらを適用ためにZodでドメインオブジェクトを表現するプラクティスを紹介しました。このプラクティスは本質的にはスキーマ定義に追加だけであり、TypeScriptのプラグラミングスタイルに大きな影響を及ぼさずに適用できる利点があります。是非検討してみてください。

# 参考記事

- [Master the Zen of the Zod Brand](https://spin.atomicobject.com/zod-brand/)
- [TypeScript の Type Branding をより便利に活用する方法のまとめ](https://zenn.dev/noshiro_piko/articles/typescript-branded-type-int)
- [Unlocking type-safety superpowers in Typescript with nominal and refinement types](https://zackoverflow.dev/writing/nominal-and-refinement-types-typescript/)
- [さらなる型安全性を求めて ~ Refinement TypeをScalaで実現する ~](https://engineering.visional.inc/blog/168/scala-refined-newtype/)
- [公称型をTypeScriptで実現するための基礎](https://qiita.com/suin/items/ae9ed911ebab48c98835)
- [TypeScript の Type Branding をより便利に活用する方法のまとめ](https://zenn.dev/noshiro_piko/articles/typescript-branded-type-int)