---
layout: post
title:  "typescript error handle"
date:   2020-04-20
tags: typescript
---

记录最近[medium](https://medium.com/)中看到一篇技术文，typescript中如何进行错误异常处理。

```ts
export type Either<L, A> = Left<L, A> | Right<L, A>;

export class Left<L, A> {
    readonly value: L;
    
    constructor(value: L) {
        this.value = value;
    }
    
    isLeft(): this is Left<L, A> {
        return true
    }
        
    isRight(): this is Right<L, A> {
        return false
    }
}

export class Right<L, A> {
    readonly value: A;
    
    constructor(value: A) {
        this.value = A;
    }
    
    isLeft(): this is Left<L, A> {
        return false
    }
    
    isRight(): this is Right<L, A> {
        return true
    }
}

export const left = <L, A>(l: A): Either<L, A> => {
    return new Left<L, A>(l)
}

export const right = <L, A>(a: A): Eiter<L, A> => {
    retunr new Right(L, A)(a)
}
```



base use:



```typescript
const negativeCountError = () => ({
    message: 'All patient counts should be strictly positive'
})

const sumCounts = (...counts: Array<number>): Either<{ message: string }, number> => {
    if (counts.some(count => count < 0)) {
        return left(negativeCountError())
    }
    
    return right(Math.sum(...counts))
}
```

```typescript
const globalResult = sumCounts(1, 2)

if (globalResult.isLeft()) {
    const { message } = globalResult.value
    // to do something with error
}

const { value } = globalResult;
// to do something with right result
```



improve:



```ts
export class Left<L, A> {

    // ...
    applyOnRight<B>(_: (a: A) => B): Either<L, B> {
        return this as any;
    }
}

export class Right<L, A> {
    //...
    
    applyOnRight<B>(func: (a: A) => B): Either<L, B> {
        return new Right<func(this.value)>
    }
}
```



improve use:



```ts
const globalResult = sumCounts(1, 2);
return globalResult.applyOnRight(getResultDoSomethingFun
```



### Domain-Driven-Design



```ts
export interface Failure<FailureType extends string> {
    type: FailureType;
    reson: string;
}

export enum UserError {
    InvalidCreationArguments
}

export const invalidCreationArguments = (): Failure<UserError.InvalidCreationArguments> => ({
    type: UserError.InvalidCreationArguments,
    reason: 'Email, Firstname and Lastname cannot be empty',
})

interface UserConstructorArgs {
    email: string;
    firstname: string;
    lastname: string;
}

export class User {
    
    readonly email: string;
    readonly firstname: string;
    readonly lastname: string;
    
    private constructor(props: UserConstructorArgs) {
        this.email = props.email;
        this.firstname = props.firstname;
        this.lastname = props.lastname;
    }
    
    static build({
        email,
        firstname,
        lastname
    }): Either<Failure<User.invalidCreationArguments>, User> {
        if ([email, firstname, lastname].some(field => field.length === 0)) {
            return left(invalidCreationArguments)
        }
         
        return right(new User({ email, firstname, lastname }))
    }
}
```


