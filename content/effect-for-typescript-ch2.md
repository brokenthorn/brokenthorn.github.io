+++
title = "Effect-TS 2: The Effect"
date = 2025-07-08T02:00:00Z
description = "Introduction to the Effect abstraction and data type from the Effect library for TypeScript."
draft = false

[extra]
keywords = "Effect, TypeScript"
series = "The Effect library for TypeScript"
toc = true

[taxonomies]
tags = [
    "The Effect library for TypeScript",
    "Effect",
    "TypeScript",
]
+++

In the previous chapter I introduced you to the [Effect](https://effect.website/)
ecosystem&mdash;toolkit if you want&mdash;of libraries for TypeScript.

In this chapter I will cover the `effect` library from Effect.

### What does effect provide?

If you look at the [API reference](https://effect-ts.github.io/effect/docs/effect) for the `effect`
library, you'll see that it has a lot of top level modules.

That is because this library provides the **core abstractions** used by all other Effect libraries,
but you won't be needing to use or know how to use more than a handful of these, maybe even just one
or two, depending on your use case.

Some of the things it provides are:

- The `Effect` data type.
- Other useful data types such as `Option` and `Either`.
- Utility functions for working with JavaScript types.
- Utility functions for working with Effect data types.
- A Clock abstraction (like a service you can inject).
- DateTime utils (replaces libs like date-fns).
- Cron utils.
- Configuration utils.
- and more.

The `effect` library provides a lot of functionality, but you do not have to use all of it.

I will be focusing on the `Effect` data type, as it is the data type that you use most often from
Effect.

### What is an Effect?

Effect is short for **side effect**. If you don't know what side effects are, expand the next
section. Otherwise, you can skip it.

#### What are side effects?

<details>
    <summary>Expand to read more about side effects</summary>

A side effect is an **observed behavior** of some function, or code, that _happens outside of the
function's local scope_, typically as a result of calling some other function that has side effects,
or as a result of direct manipulation of global state&mdash;state that comes from outside the
function (a variable, a database client, a file handle, etc.).

By contrast, a pure function is a function that only takes some input and produces some output, with
no other observed behavior that reaches outside the function or that depends on anything outside the
function or even outside our program (like a network socket for example).

Pure functions return the same output, given the same input, no matter how many times you run them,
or at what instant in time.

For example, a side effectful function could create, or accept, a file handle that it then uses in
some way. Because the file handle is **global state** (state outside our function) and operating on
it _has side effects_, it makes our function have side effects as well.

If our function writes to the function, let's say it appends some text, that becomes a side effect
of running our function. If we run the function twice, it would append the same data twice. Perhaps
not what we wanted.

A side effectful function is also any function that throws an exception, because it alters the
normal flow of program execution. After all, `throw` is just a glorified `goto`. Throwing exceptions
is very problematic in concurrent or async code, and this is where Effect's
[structured concurrency](https://en.wikipedia.org/wiki/Structured_concurrency) shines. It makes it
possible to handle **all** errors where they happen or for all errors to be propagated back to the
control flow of the parent scope. With `throw` and `catch` you have no way of telling what error
you're handling, or where each error is handled, if at all.

Some programming languages have
[checked exceptions](<https://en.wikipedia.org/wiki/Exception_handling_(programming)#Checked_exceptions>),
but we won't get into that. While they're an upgrade in safety compared to unchecked exceptions
(like in JavaScript), they're still inferior to Effects.

</details>

#### The Effect data type

The `Effect` data type is an abstraction for expressing computations, both synchronous or
asynchronous, their return values, the errors they produce, the requirements they have, and at the
same time, make it easier to understand if they have side effects or not.

`Effect`s are lazy: they describes some computation, but they don't execute it when an Effect is
created.

An `Effect` describes a computation using 3 type parameters, 2 of them defaulting to `never`, if not
specified: `Effect<T, E=never, R=never>`.

- `T`: is the return **T**ype, also called **the success channel**.
- `E`: is the **E**rror type, also called **the error channel**.
- `R`: is the **R**equirements type, also called **the requirements channel**.

The simplest `Effect` you can express is an effect that produces a value of type `T` and has no
errors or requirements: an `Effect<T>`. This is similar to a `Promise<T>`&mdash;at least with
regards to `T`&mdash;but Promises are not lazy, they start executing immediately after they're
created, and can throw unforeseen errors.

Because creating Effects has no side effects, they're just just values, they can be composed using
functional combinators like `flatMap`, `zip`, or `gen`, before they are executed.

They can also be transformed using `map`, `mapError` or `mapBoth`, which change only the success
channel, only the error channel, or both.

This composability is what makes Effects superior for building complex and robust applications,
especially when dealing with concurrent code.

## What advantages does an Effect provide?

Effects have many advantages that stem from their declarative nature and ability to track their
observable behavior using the type system.

I will focus on the type signature and this declarative nature to give you some examples of the
advantages `Effect` brings.

### No more exceptions (errors as values)

Consider this normal function:

```typescript
const divide = (a: number, b: number): number => {
  if (b === 0) {
    throw new Error("Cannot divide by zero");
  }
  return a / b;
};
```

If you just glance at this function's type you'd never guess it could completely crash and burn if
you tried to divide by zero. The only way to know that is to actually read through the code.

This isn't a huge deal when you're just starting out with a few functions. But imagine your project
gets bigger, and you have tons of functions. It's super easy to forget that one of them might throw
an error, and then you also forget to plan for how to handle it.

If you're calling third party code you might also have no idea that you're calling code that might
throw and break your own code.

After your codebase grows and you get into a few unhandled exception cases, you lose trust in your
codebase because you have no idea where the errors are. So your immediate solution is to play
Pokemon..., you know, "catch 'em all", which leads to poor error handling and reporting. It also
makes it very difficult to start adding error recovery.

Now consider the alternative version, written with Effect:

```typescript
import { Effect } from "effect";

const divide = (a: number, b: number): Effect.Effect<number, Error> => {
  if (b === 0) return Effect.fail(new Error("Cannot divide by zero"));
  return Effect.succeed(a / b);
};
```

The return type is optional, because it can be inferred, but it shows you that this function now can
either succeed with a number or fail with an Error.

So, with Effect, you can use the type system to more precisely express what your code does.

### Dependencies are explicit

Remember the third type parameter, `R`? It allows you express any requirements that an Effect has,
which have to be provided before the Effect can be executed. Because this is tracked in the type
system, it's a type error if you try to execute an Effect that hasn't had its requirements removed
from the type signature by providing them before the Effect is run or before the program is started.

The Effect's requirements are a form of Dependency Injection (DI), but it's more explicit, more
flexible, is inferrable (you don't have to declare the requirements, they're inferred from use), and
it doesn't pollute your function's arguments list.

Let's consider the classical form of DI:

```typescript
// Define an abstraction for data fetching:
interface UserRepository {
  getUserName(userId: string): Promise<string>;
}

// Concrete implementation using HTTP:
class HttpUserRepository implements UserRepository {
  async getUserName(userId: string): Promise<string> {
    return Promise.resolve(`User_${userId}`);
  }
}

// Business logic that depends on the abstraction:
class UserService {
  constructor(private userRepo: UserRepository) {}

  async greetUser(userId: string): Promise<string> {
    return `Hello, ${await this.userRepo.getUserName(userId)}!`;
  }
}

// Usage (approximately):
const repository = new HttpUserRepository();
const service = new UserService(repository);

service.greetUser("42").then(console.log);
// Hello, User_42!
```

The usage demo is greatly simplified, as you would normally have to configure a DI container and set
up which `UserRepository` is created, and have to set up some wiring so that the correct instance is
injected for you.

The alternative Effectful version looks like this:

```typescript
import { Context, Effect, Layer } from "effect";

// Define the service interface as a Context Tag:
export class UserRepository extends Context.Tag("UserRepository")<
  UserRepository,
  { getUserName(userId: string): Effect.Effect<string> } // our interface
>() {}

// Provide the implementation as a Layer (it describes how a Tag is constructed):
export const UserRepositoryLive: Layer.Layer<UserRepository> = Layer.succeed(UserRepository, {
  getUserName: (userId) => Effect.succeed(`User_${userId}`),
});

// Business logic as a pure Effect (the return type can be inferred and is optional):
const greetUser = (userId: string): Effect.Effect<string, never, UserRepository> =>
  Effect.gen(function* () {
    const repository = yield* UserRepository; // auto adds UserRepository to the Requirements type param
    return `Hello, ${yield* repository.getUserName(userId)}!`;
  });

// Usage (this is it, nothing more required):
Effect.runPromise(greetUser("42").pipe(Effect.provide(UserRepositoryLive))).then(console.log);
// Hello, User_42!
```

<details>
  <summary>Why is Effect using generators (`function*`)?</summary>

We'll ignore the use of a generator function (`function*`) for now, but if you want to understand
why Effect uses them, you can
[read this documentation](https://effect.website/docs/getting-started/using-generators/).

If you think they are slow, [they are not](https://effect.website/docs/additional-resources/myths/).
That would be like saying async/await it slow.

</details>

You're seeing a few new types:

- `Context.Tag`, which is a tag for uniquely identifying something from a `Context`, which is like a
  DI Container (but you can any number of them in one app). It identifies not just what a generic
  thing is called but also its interface or type.
- `Layer`, which is an Effect for constructing `Context.Tag`, so you could call it a factory.

While the usage looks a lot like the one for classical DI above, in this case this is all you need
to do, there's no other setup or wiring required, like with classical DI. This is more flexible, not
just more explicit.

You could lazily decide what concrete implementation to provide by creating a dynamic Layer with
`Layer.effect(tag, eff)`. The building and decision happens in the `eff` Effect. For example, you
could allow the Layer construction to be configured from the process environment (Effect offers the
`Config` module for this).

Because Layers are just Effects (their type signatures are identical), they can also return errors
if a dependency could not be constructed. They could also declare requirements/dependencies of
themselves.

`Layer.effect` is also very useful when you have a service that is slow to build or requires to be
built asynchronously (like a database connection pool).

Here is a quick side by side comparison between classical DI and Effect:

| Aspect             | Dependency Injection                                                                          | Effect                                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Goal               | Manage dependencies and side effects via interfaces and complicated wiring with DI Containers | Track and control side effects explicitly in types                                            |
| Side effects       | Hidden behind abstractions (interface, exceptions)                                            | Explicitly declared and handled (errors, requirements)                                        |
| Coupling           | Loose coupling via interfaces                                                                 | Loose coupling via effect handlers/combinators (e.g., error handlers, functional combinators) |
| Error handling     | Usually separate, often exceptions                                                            | Often integrated as part of effects or effect pipeline                                        |
| Runtime vs Compile | Mostly runtime wiring                                                                         | Mostly compile-time effect tracking (better static analysis)                                  |
| Complexity         | Simpler to adopt, but can get complex in large apps                                           | More complex initially, but safer and more explicit                                           |
| Use cases          | General OOP and modular design                                                                | Functional programming, pure functional cores                                                 |

## Why is Effect better?

The Effect-based approach offers several advantages over traditional DI:

1. **Explicitness:** Side-effects and dependencies are explicitly declared in the type system,
   making them easier to reason about and track.
2. **Composability:** Effects can be combined and transformed using functional combinators, enabling
   modular and reusable code.
3. **Unified Abstraction:** Error handling, async operations, state, and other concerns are unified
   under the `Effect` abstraction, reducing the need for separate mechanisms.
4. **Safety:** The type system enforces handling of effects or propagates them explicitly to the
   layers above, reducing the risk of runtime errors or unhandled side effects (exceptions).
5. **Testability:** Effects are pure descriptions until executed, making them easy to test by
   substituting requirements with mocks.

While the classical DI approach works well for many use cases, it hides side effects behind
abstractions. For example, can I see from the function signature if `getUserName()` calls the
network or opens a database connection? If I don't see a specific interface in its parameters and
know how the DI container is set up, then no. Likewise, do I know whether it can throw an exception
and break normal program flow without reading its code? No.

Classical DI relies on runtime wiring, which can lead to subtle bugs and harder debugging.

Effect, by contrast, leverages the type system and makes everything more explicit and safe. You have
more control over the executing code and you get stronger guarantees about your program's behavior.

## Conclusion

Managing challenges like error handling, debugging, tracing, async/promises, retries, streaming,
concurrency, caching, resource management, and a lot more are made manageable with Effect.

We'll dive into more depth in some of these topics in the next articles, but suffice to say, you
don’t have to re-invent the solutions to these problems, or install tons of dependencies. Effect,
under one umbrella, solves many of the problems that you would usually install many different
dependencies with different APIs to solve.

Effect is heavily inspired by great work done in other languages, like Scala and Haskell. However,
it’s important to understand that Effect’s goal is to be a practical toolkit, and it goes to great
lengths to solve real, everyday problems that developers face when building applications and
libraries in TypeScript.

## Next Steps

In the next chapter I will focus more on practical examples to show you why Effects are better than
traditional approaches. I will introduce errors and error handling with Effect, comparing a
`Promise<T>`, which doesn't tell you if it can throw an error or not, with `Effect<T, SomeError>`.
