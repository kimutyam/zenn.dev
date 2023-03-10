---
title: "zod"
emoji: "ð"
type: "tech" # tech: æè¡è¨äº / idea: ã¢ã¤ãã¢
topics: ["zod", "fp-ts"]
published: false
---

## airticle prot

- å¨ä½ãéãã¦è§£æ±ºããããã¨
- zodãè§£æ±ºãããã¨
- fp-tsãè§£æ±ºãããã¨
- zodã¨fp-tsãçµã¿åããã
- ä»æ§ãã¿ã¼ã³..?
- (å¿ç¨) NestJSã¨çµã¿åããã


## test

TypeScriptã§åå®å¨æ§ãå®ç¾ããéç¨ã§`fp-ts`ã®æ´¾çã©ã¤ãã©ãªã§ãã`newtype-ts`ãçºè¦ãã¾ãããTypeScriptã®åã·ã¹ãã ã¯æ§é çé¨åå(Structural Subtyping)ã§ããã`newtype-ts` ãå©ç¨ããã¨ç°¡æã«å¬ç§°å(NewType)ãæã«å¥ãããã¨ãã§ãã¾ããããã«`newtype-ts` ã® `Prism` ãä½¿ããã¨ã§å¬ç§°åã«åå¶ç´ãè¨­ããå¶ç´éåãªã¤ã³ã¹ã¿ã³ã¹åãã§ããªãããã«ãããã¨ãã§ãã¾ããããã¯å¥ç´ãã­ã°ã©ãã³ã°ã§ããã¨ããã®ä¸å¤æ¡ä»¶ã¨ãã¦æ±ãã¾ããè©³ããã¯ä»¥ä¸ã®è¨äºã§ç´¹ä»ãã¦ãã¾ãããæ¬ç¨¿ã§ã¯å¿é ã®ç¥è­ã«ãªãã¾ãã

https://kimutyam.medium.com/newtype-ts-iso-prism-15b65414889d

ãã¦ãã·ã¹ãã å¤é¨ããåãåã£ãå¤(ä»¥éãå¤é¨å¤ã¨ãã)ãããªãã¼ã·ã§ã³ãNewTypeã«å¤æãããã¨ãæ³åãã¦ã¿ã¾ãããããã®æã«ããã¤ãã®åé¡ã«ã¶ã¡å½ããã¾ãããã®åé¡ãè§£æ¶ããããã«zodãå°å¥ããzodã¹ã­ã¼ãããNewTypeã«å¤æããã¢ãã­ã¼ããç´¹ä»ãã¾ãã(ç²ç®)

--

1. å¤é¨å¤ã`Prism#getOption` ã«å¥åãã¦ã¿ã¾ãããã

:::details ã¿ã¤ãã«

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
      id: liftLeft(UserId.prism.getOption)((x) => `ã¦ã¼ã¶ã¼IDã¯UUIDå½¢å¼ã§å¥åãã ãã: ${x}`)(
        rawUserId,
      ),
      name: liftLeft(UserName.prism.getOption)((x) => `ã¦ã¼ã¶ã¼åã¯4æå­ä»¥ä¸ã§å¥åãã ãã: ${x}`)(
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


ãã®æã®åé¡ã¯

- ããªãã¼ã·ã§ã³ã­ã¸ãã¯ããã¤ã©ã¼ãã¬ã¼ãåãã¦ããç¹
- ããªãã¼ã·ã§ã³ã¨ã©ã¼ãªãã¸ã§ã¯ããEither#Leftããçæããå¿è¦ãããç¹

ããªãã¼ã·ã§ã³ã®ã¨ã³ã·ã¹ãã ãæã£ã¦ããzodãä½¿ãã°ãã®è¾ºã¯è§£æ¶ãã¾ãã


2. å¤é¨å¤ãzodã«parseããå¾ã«

- ããªãã¼ã·ã§ã³ã­ã¸ãã¯ã¨NewTypeã®åå¶ç´ãéè¤ãã¦ãã¾ãã

3. zod.brandãä½¿ã

- fp-tsã»ã©ã®åå®å¨æ§ã¯ãªãããã§ãã




> Typescript ã§å®å¨ãªåå®å¨æ§ãå®ç¾ãããã¨ããæã®ä¸­ã§ãç§ã¯ newtype-ts ã«åºä¼ãã¾ããã
> ãã®ã©ã¤ãã©ãªã¯åå®å¨ãªãã©ã³ãå(newtypes, opaque types, nominal types ã¨ãå¼ã°ãã)ãå®è£ãããã¨ãã§ããã
> åæã«ãç§ã¯ãã¼ã¿ã®åãå®è¡æã«æ¤è¨¼ããããã«zodãä½¿ç¨ãã¦ãã¾ã(ããã¦ãããæ¨å¥¨ãã¦ãã¾ã)ã


> Zodã®ç¹å¾´ã¨ãã¦ãSchema firstãªvalidationã©ã¤ãã©ãªã§ããã¨ããã®ãããã¾ãã
validateããschemaï¼åä¸ã®schemaããobject, arrayã¾ã§ï¼ãå®ç¾©ããããããã¼ã¹ã«parseããã¨ãããã®ã§ãã


```typescript
import { prism } from 'newtype-ts';
import type { Newtype } from 'newtype-ts';
import { z } from 'zod';
import { unsafeNew } from '@sample/newTypeFactory';
import type { NewTypeFactory } from '@sample/newTypeFactory';

/** ã¦ã¼ã¶ã¼å */
export type UserName = Newtype<{ readonly USER_NAME: unique symbol }, string>;

const MIN = 1;
const MAX = 20;

const errorMessage = `ååã¯${MIN}æå­ä»¥ä¸${MAX}æå­ä»¥åã§æå®ãã¦ãã ãã`;

const zodTypeWithoutTransform = z
  .string()
  .min(MIN, errorMessage)
  .max(MAX, errorMessage)
  .describe('ã¦ã¼ã¶ã¼å');

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
 * NewType(å¬ç§°å)ã®ãã¡ã¯ããªã¤ã³ã¿ã¼ãã§ã¼ã¹
 *
 * zodã®ã©ã³ã¿ã¤ã ã®åä»ã¨newtype-tsã®å¬ç§°åãå±å­ãããããã«ãä»¥ä¸ã®ãªã³ã¯ãåèã«ãã¾ããã
 */
export interface NewTypeFactory<S, T> {
  prism: Prism<S, T>;
  zodType: z.ZodEffects<z.ZodTypeAny, T, S>;
  unsafeNew: (raw: S) => T;
}
```

refineãä½¿ããªãã®ã¯ããªãã¼ã·ã§ã³




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

TODO: Moduleã®ãµã³ãã«ã³ã¼ããä½æ

```typescript
```

TODO: Swaggerã®ãµã³ãã«ã³ã¼ããä½æ
