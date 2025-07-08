+++
title = "Intro to Effect: Chapter 2 - The effect package"
date = 2025-07-08T02:00:00Z
description = "The second chapter in a series on Effect. Introduces the Effect abstraction."
draft = false

[extra]
keywords = "Effect, TypeScript"
series = "Intro to Effect"
toc = true

[taxonomies]
tags = [
    "Intro to Effect",
    "Effect",
    "TypeScript",
]
+++

# Introduction

In the previous chapter, I introduced Effect and why you might want to use it
(or not). Effect is a collection of libraries&mdash;a toolkit. One key package
is the core package, simply named `effect`. This chapter provides a high-level
overview of that package.

# The effect Package

The `effect` package provides the core abstraction used by all other Effect
libraries and by you to write Effect-ful programs. These programs use the
`Effect` abstraction to compose computations in a structured and predictable
way.

## What is an Effect?

At its core, an `Effect` is a data type that represents a computation, similar
to a `Promise`. However, unlike Promises, Effects are _lazy_&mdash;they do not
execute until explicitly run. Effects are also more general, capable of
modeling synchronous computations, errors, resource management, and
asynchronous operations.

Effects can be composed using functional combinators like `flatMap`, `zip`, or
`gen`. This composability allows you to build complex workflows from smaller,
reusable pieces.

## Effects and side-effects

Effects are designed to model side-effects explicitly. Side-effects include
actions like state changes, exceptions, I/O operations, and time-dependent
computations. By representing these as `Effect`s, you make them explicit in the
type system, improving reasoning, safety, and testability.

For example, a function that prints to the console or fetches data from a
network produces side-effects. Without an abstraction like `Effect`, these
actions are implicit and harder to manage. By contrast, `Effect` makes
side-effects explicit, ensuring they are predictable and occur only when
intended.

## Effect Systems

Effect Systems extend the type system to track side-effects explicitly.
Functions declare the effects they perform (e.g., reading a file, making a
network call) in their type signatures. The type system ensures these effects
are either handled (e.g., by providing required dependencies or catching
errors) or propagated to the caller. This guarantees that no effect is ignored
or hidden.

Effect Systems unify concerns like error handling, async operations, state, and
logging into a single, composable abstraction. They encourage a pure functional
core with clear effect boundaries, reducing the risk of accidental
side-effects. While they introduce a steeper learning curve and some
boilerplate, the benefits include improved safety, composability, and
maintainability.

# Effects vs. Dependency Injection: A Code Comparison

To illustrate the differences, let’s compare a traditional Dependency Injection
(DI) approach with an Effect-based approach.

First, a brief comparison:

| Aspect             | Dependency Injection & Abstractions                 | Effect Systems                                      |
| ------------------ | --------------------------------------------------- | --------------------------------------------------- |
| Goal               | Manage dependencies and side effects via interfaces | Track and control side effects explicitly in types  |
| Side effects       | Hidden behind abstractions                          | Explicitly declared and handled                     |
| Coupling           | Loose coupling via interfaces                       | Loose coupling via effect handlers/combinators      |
| Error handling     | Usually separate, often exceptions                  | Often integrated as part of effects                 |
| Runtime vs Compile | Mostly runtime wiring                               | Mostly compile-time effect tracking                 |
| Complexity         | Simpler to adopt, but can get complex in large apps | More complex initially, but safer and more explicit |
| Use cases          | General OOP and modular design                      | Functional programming, pure functional cores       |

## Dependency Injection Example

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

// Business logic depends on the abstraction:
class UserService {
  constructor(private userRepo: UserRepository) {}

  async greetUser(userId: string): Promise<string> {
    const name = await this.userRepo.getUserName(userId);
    return `Hello, ${name}!`;
  }
}

// Usage:
const userRepo = new HttpUserRepository();
const userService = new UserService(userRepo);

userService.greetUser("42").then(console.log); // Hello, User_42!
```

## Effect System Example

```typescript
import { Context, Effect, Layer } from "effect";

// Define the service interface as a Context.Tag:
export class UserRepository extends Context.Tag("UserRepository")<
  UserRepository,
  { getUserName(userId: string): Effect.Effect<string> }
>() {}

// Provide the implementation as a Layer:
export const UserRepositoryLive: Layer.Layer<UserRepository> = Layer.succeed(
  UserRepository,
  {
    getUserName: (userId) => Effect.succeed(`User_${userId}`),
  },
);

// Business logic as a pure Effect:
const greetUser = (
  userId: string,
): Effect.Effect<string, never, UserRepository> =>
  Effect.gen(function* () {
    const repository = yield* UserRepository;
    const name: string = yield* repository.getUserName(userId);
    return `Hello, ${name}!`;
  });

// Note the return type of greetUser. The error is never, which means either
// there is no error or that any errors that might happen in the composed
// effects or inside Effect.gen, the Layer constructor (yield* UserRepository), or
// in `getUserName`, must be handled before returning the composed Effect, or
// outside this code, in the other Effects' constructors. Thus
// errors/exceptions are explicit side-effects.

// Run the Effect:
Effect.runPromise(
  greetUser("42").pipe(Effect.provide(UserRepositoryLive)),
).then(console.log); // Hello, User_42!
```

## Why the Effect-Based Approach is Better

The Effect-based approach offers several advantages over traditional DI:

1. **Explicitness:** Side-effects and dependencies are explicitly declared in the type system, making them easier to reason about and track.
2. **Composability:** Effects can be combined and transformed using functional combinators, enabling modular and reusable code.
3. **Unified Abstraction:** Error handling, async operations, state, and other concerns are unified under the `Effect` abstraction, reducing the need for separate mechanisms.
4. **Safety:** The type system enforces handling or propagation of effects, reducing the risk of runtime errors or unhandled side-effects.
5. **Testability:** Effects are pure descriptions until executed, making them easy to test by substituting environments or mocking dependencies.

While the DI approach works well for many use cases, it hides side-effects
behind abstractions (e.g., do I know from the types if `getUserName()` calls
the network or not) and relies on runtime wiring, which can lead to subtle bugs
and harder-to-test code. The Effect-based approach, by contrast, makes
side-effects explicit, composable, and type-safe, leading to more predictable
and maintainable systems.

# Conclusion

The `effect` package provides a powerful abstraction for managing side-effects
in a functional and composable way. By making side-effects explicit in the type
system, Effect Systems improve reasoning, safety, and testability. While they
introduce some complexity, the benefits of explicitness, composability, and
unification make them a compelling choice for building robust and maintainable
systems.

# Next Steps

In the next chapter I will focus more on examples to show you why Effects are
better than traditional approaches. I will introduce errors and error handling
with Effect, comparing a `Promise<T>`, which doesn't tell you if it can throw
an error or not, with `Effect<T, SomeError>`, which does.

Imagine writing a Fetch call and how many throws and catches you have to write
as your code for interacting with your backend evolves with time. And that is
only if you want to create a clean and granular error handling strategy that
makes it possible to show your users error messages that are actually useful
and less annoying.

With Effect you don't have to throw ever again.
