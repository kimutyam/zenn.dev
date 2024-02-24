---
title: "zod"
emoji: "🍏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zod", "fp-ts"]
published: false
---

## airticle prot

- 全体を通して解決したいこと
- zodが解決すること
- fp-tsが解決すること
- zodとfp-tsを組み合わせる
- 仕様パターン..?
- (応用) NestJSと組み合わせる


## はじめに

TypeScriptで型安全性を実現する過程で`fp-ts`の派生ライブラリである`newtype-ts`を発見しました。
TypeScriptの型システムは構造的部分型(Structural Subtyping)ですが、`newtype-ts` を利用すると簡易に公称型(NewType)を手に入れられます。
さらに`newtype-ts`の`Prism`を使うことで公称型に型制約を設け、制約違反なインスタンスを生成できないように制御ができます。
詳しくは以下の記事で紹介していますが、本稿では必須の知識になります。

https://kimutyam.medium.com/newtype-ts-iso-prism-15b65414889d

## 前提

ユーザーアカウントの登録

```typescript
import type { Newtype } from 'newtype-ts';
import { prism } from 'newtype-ts';
import { validate as uuidValidate } from 'uuid';

export type UserId = Newtype<{ readonly USER_ID: unique symbol }, string>;
export const UserId = {
  prism: prism<UserId>((id) => uuidValidate(id)),
} as const;
```

```typescript
import type { Newtype } from 'newtype-ts';
import { prism } from 'newtype-ts';

export type UserName = Newtype<{ readonly USER_NAME: unique symbol }, string>;
export const UserName = {
  prism: prism<UserName>((name) => name.length >= 4),
} as const;
```

```typescript
import { UserId } from './userId';
import { UserName } from './userName';

export type User = Readonly<{
  id: UserId;
  name: UserName;
}>;
```


さて、システム外部から受け取った値(以降、外部値とする)をバリデーションしNewTypeに変換することを想像してみましょう。
この時にいくつかの問題にぶち当たります。その問題を解消するためにzodを導入し、zodスキーマからNewTypeに変換するアプローチを紹介します。(粗目)




--

1. 外部値を`Prism#getOption` に入力してみましょう。

:::details タイトル


```typescript
import * as AP from 'fp-ts/Apply';
import * as E from 'fp-ts/Either';
import * as NA from 'fp-ts/NonEmptyArray';
import { liftLeft } from '../../../util/liftLeft';
import { UserId } from './userId';
import { UserName } from './userName';

export type User = Readonly<{
  id: UserId;
  name: UserName;
}>;

export const User = {
  decode: (rawUserId: string, rawUserName: string): E.Either<NA.NonEmptyArray<string>, User> =>
    AP.sequenceS(E.getApplicativeValidation(NA.getSemigroup<string>()))({
      id: liftLeft(UserId.prism.getOption)((x) => `ユーザーIDはUUID形式で入力ください: ${x}`)(
        rawUserId,
      ),
      name: liftLeft(UserName.prism.getOption)((x) => `ユーザー名は4文字以上で入力ください: ${x}`)(
        rawUserName,
      ),
    }),
} as const;
```

```typescript
import * as E from 'fp-ts/Either';
import { pipe } from 'fp-ts/function';
import type * as NA from 'fp-ts/NonEmptyArray';
import type { Option } from 'fp-ts/Option';

export const liftLeft =
  <S, A>(getOption: (s: S) => Option<A>) =>
  <L>(toLeftF: (ss: S) => L): ((aa: S) => E.Either<NA.NonEmptyArray<L>, A>) =>
  (x) =>
    pipe(
      getOption(x),
      E.fromOption(() => toLeftF(x)),
      E.mapLeft((a) => [a]),
    );

```
:::


この時の問題は

- バリデーションロジックがボイラープレート化している点
- バリデーションエラーオブジェクトをEither#Leftから生成する必要がある点

バリデーションのエコシステムを持っているzodを使えばこの辺は解消します。


2. 外部値をzodにparseした後に

- バリデーションロジックとNewTypeの型制約が重複しています。

3. zod.brandを使う

- fp-tsほどの型安全性はないようです。




> Typescript で完全な型安全性を実現しようとする旅の中で、私は newtype-ts に出会いました。
> このライブラリは型安全なブランド型(newtypes, opaque types, nominal types とも呼ばれる)を実装することができる。
> 同時に、私はデータの型を実行時に検証するためにzodを使用しています(そしてそれを推奨しています)。


> Zodの特徴として、Schema firstなvalidationライブラリであるというのがあります。
validateするschema（単一のschemaからobject, arrayまで）を定義し、それをベースにparseするというものです。


```typescript
import { prism } from 'newtype-ts';
import type { Newtype } from 'newtype-ts';
import { z } from 'zod';
import { unsafeNew } from '@sample/newTypeFactory';
import type { NewTypeFactory } from '@sample/newTypeFactory';

/** ユーザー名 */
export type UserName = Newtype<{ readonly USER_NAME: unique symbol }, string>;

const MIN = 1;
const MAX = 20;

const errorMessage = `名前は${MIN}文字以上${MAX}文字以内で指定してください`;

const zodTypeWithoutTransform = z
  .string()
  .min(MIN, errorMessage)
  .max(MAX, errorMessage)
  .describe('ユーザー名');

const prismUserName = prism<UserName>((x) => zodTypeWithoutTransform.safeParse(x).success);

const unsafeNewUserName = (raw: string): TenantName =>
  unsafeNew(errorMessage)(prismUserName)(raw);

const factory: NewTypeFactory<string, TenantName> = {
  prism: prismUserName,
  zodType: zodTypeWithoutTransform.transform(unsafeNewTenantName),
  unsafeNew: unsafeNewUserName,
};

export const UserName = {
  factory,
} as const;
```

```typescript
import assert from 'assert';
import * as O from 'fp-ts/Option';
import type { Prism } from 'monocle-ts';
import type { z } from 'zod';

export const unsafeNew =
  (errorMessage: string) =>
  <S, T>(prism: { getOption: (s: S) => O.Option<T> }) =>
  (value: S): T => {
    const opt = prism.getOption(value);
    assert(O.isSome(opt), errorMessage);
    return opt.value;
  };

/**
 * NewType(公称型)のファクトリインターフェース
 *
 * zodのランタイムの型付とnewtype-tsの公称型を共存させるために、以下のリンクを参考にしました。
 */
export interface NewTypeFactory<S, T> {
  prism: Prism<S, T>;
  zodType: z.ZodEffects<z.ZodTypeAny, T, S>;
  unsafeNew: (raw: S) => T;
}
```

refineを使わないのはバリデーション




```typescript
import { createZodDto } from 'nestjs-zod';
import { z } from 'zod';
import { UserId } from '@sample/userId';

export class GetParameter extends createZodDto(
  z.object({
    userId: UserId.factory.zodType,
  }),
) {}

```

```typescript
import { createZodDto } from 'nestjs-zod';
import { z } from 'zod';
import { UserId } from '@sample/userId';
import { UserName } from '@sample/userName';

export class TenantRoleGetResponse extends createZodDto(
  z.object({
    roleGroup: RoleGroup.zodType,
  }),
) {}
```

```typescript
import { APP_PIPE } from '@nestjs/core';
import { ZodValidationPipe } from 'nestjs-zod';

export const ZodValidationProvider = {
  // SEE: https://github.com/risenforces/nestjs-zod#globally-recommended
  provide: APP_PIPE,
  useClass: ZodValidationPipe,
};
```

TODO: Moduleのサンプルコードを作成

```typescript
```

TODO: Swaggerのサンプルコードを作成
