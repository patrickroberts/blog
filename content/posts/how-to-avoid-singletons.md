+++
title = 'How to Avoid Singletons'
date = 2025-03-02
draft = false
+++
## Why are Singletons so Painful?

People often say that singletons are "bad" and "evil", and while I am sympathetic to the criticism, I find that these sort of dogmatic assertions are too vague; they don't articulate the true nature of the pattern. Singletons are just *painful*. Even though I don't personally believe there's anything inherently "bad" or "evil" about singletons (or the developers that resort to using them, for that matter), I do agree that their usage leads to many avoidable headaches. Today I'm going to share one possible solution for avoiding singletons in C++, with inspiration from [React](https://react.dev) of all places.

### Static Lifetimes are Difficult to Reason About

Singletons, which are static storage duration objects, are constructed before the program's control flow enters `main` and their [static initialization order is not well-defined](https://isocpp.org/wiki/faq/ctors#static-init-order) (except for [static block variables](https://en.cppreference.com/w/cpp/language/storage_duration#Static_block_variables) since C++11, which are initialized the first time control flow passes through their declaration). They are also not destructed until after `main` exits, and their destructor will not be invoked at all if `main` throws an exception, causing the program to abort. In contrast, invocation of automatic storage duration destructors can be ensured by declaring them in the scope of a `try` block, as long as the program doesn't *immediately* abort (e.g. by throwing inside of a `noexcept` function). This is true even if the corresponding `catch` block re-`throw`s the exception and causes the program to abort. Because of this, by comparison singletons are a poorly suited pattern for [resource acquisition](https://en.cppreference.com/w/cpp/language/raii).

### Mutable Access Requires Synchronization

If a singleton can be mutated from multiple threads of execution, access must at least be performed atomically to have well-defined behavior, and often additionally requires synchronization if any particular modification order needs to be enforced. This limitation can be worked around in certain cases that are still satisfied by using the [`thread_local` storage class specifier](https://en.cppreference.com/w/cpp/language/storage_duration), though doing so might technically disqualify it as a true singleton. (but honestly, who cares?)

### Unit Tests Become Stateful

Arguably one of the most egregious lapses of sanity when using singletons is that unit tests running in a single process no longer behave independently of each other by default. Setup and/or teardown of singleton instances must be explicitly performed between unit tests to ensure that the outcome of one test is not influenced by previously invoked side-effects in others. Because of this, singletons are often one of the first suspects investigated when troubleshooting failing unit tests that seem unrelated to pending changes. One of the worst scenarios is when a test only ever passed because of singleton state reached as a side-effect of another test. This can be somewhat mitigated by [red-green testing](https://stackoverflow.com/a/276824), but unfortunately refactoring can also occasionally cause tests to start succeeding for the wrong reasons. The following example shows how singletons can cause this to occur in tests.

```cpp
#include <functional>
#include <optional>

#include "gtest/gtest.h"

bool is_enabled = false;

void set_enabled(bool state) { is_enabled = state; }

bool get_enabled() { return is_enabled; }

void dispatch(int event, std::function<void(int)> callback) {
  if (get_enabled()) {
    callback(event);
  }
}

TEST(Dispatcher, DropsByDefault) {
  auto actual = std::optional<int>{};
  const auto expected = std::nullopt;

  // default should be disabled
  dispatch(99, [&](int result) { actual = result; });

  EXPECT_EQ(actual, expected);
}

TEST(Dispatcher, ForwardsWhenEnabled) {
  auto actual = std::optional<int>{};
  const auto expected = 42;

  set_enabled(true);
  dispatch(expected, [&](int result) { actual = result; });

  EXPECT_EQ(actual, expected);
}
```

At first glance, this might seem like a reasonable test suite, but when [running with `--gtest_shuffle`](http://google.github.io/googletest/advanced.html#shuffling-the-tests), `DropsByDefault` has a 50% chance of failure (the case when `ForwardsWhenEnabled` runs first) due to these tests unintentionally interacting through the static variable `is_enabled`. Firstly, it would be advisable to replace the comment `// default should be disabled` in `DropsByDefault` with `ASSERT_FALSE(get_enabled());` as a sanity check, since doing so would help make this potential failure much easier to debug. One possible resolution though would be to make the following change to `ForwardsWhenEnabled`.

```cpp
TEST(Dispatcher, ForwardsWhenEnabled) {
  auto actual = std::optional<int>{};
  const auto expected = 42;
  // cache previous state
  const auto previous = get_enabled();

  set_enabled(true);
  dispatch(expected, [&](int result) { actual = result; });

  EXPECT_EQ(actual, expected);

  // restore previous state
  set_enabled(previous);
}
```

This ensures that `DropsByDefault` always uses the default behavior regardless of test order by having `ForwardsWhenEnabled` restore it before it returns.

---

## Singletons are a Temptation

At this point, you might be thinking:

> My singleton has a trivial destructor, my code is single-threaded, and all my tests are run in isolation to prevent unintended interactions between them.

If your code has any users at all, having a singleton in your implementation can still cause many headaches. But putting that aside for a moment, what qualities do singletons have that are so appealing to developers? Consider the following code.

```cpp
#include <algorithm>
#include <concepts>
#include <execution>
#include <ranges>
#include <variant>

using execution_policy_t = std::variant<
    std::execution::sequenced_policy,
    std::execution::parallel_policy,
    std::execution::parallel_unsequenced_policy,
    std::execution::unsequenced_policy>;

execution_policy_t execution_policy{};

void set_policy(execution_policy_t policy) {
  execution_policy = policy;
}

execution_policy_t get_policy() { return execution_policy; }

void for_each_with_policy(
    std::ranges::common_range auto &&range,
    auto func) {
  std::visit([&](auto policy) {
    std::for_each(
        policy,
        std::ranges::begin(range),
        std::ranges::end(range),
        std::move(func));
  }, get_policy());
}
```

The convenience is that once an execution policy is set by the user, the utility function `for_each_with_policy` *does not require an additional parameter*. Of course, this could easily be implemented without a singleton.

```cpp
#include <algorithm>
#include <execution>
#include <ranges>

template <class ExecPolicyT>
  requires std::is_execution_policy_v<ExecPolicyT>
auto make_for_each_with_policy(ExecPolicyT policy) {
  return [=](std::ranges::common_range auto &&range, auto func) {
    std::for_each(
        policy,
        std::ranges::begin(range),
        std::ranges::end(range),
        std::move(func));
  };
}
```

However, for the sake of exploration, we'll refer back to this initial approach using the singleton pattern later on.

To try and generalize the convenience that singletons provide, I'd probably say something like

> A singleton allows an algorithm to use a dependency without an additional parameter.

It's reasonable to question the validity of this desire in the first place, but at some point we must acknowledge that the temptation to use singletons is inevitable when inexperienced developers are given a deadline for adding new features to existing code. It takes precious time to modify an interface that's widely used throughout a codebase. Sometimes it's even challenging just to accept when a current design is no longer sufficient for a new feature that is needed.

The biggest hurdle of all is implementing a design that will be able to meet the requirements of all anticipated features without over-generalizing. A lot of experience is needed to find the balance between a generic design that's so abstract it's only marginally useful, and a pragmatic design that's so tailored to one use-case it can't be cleanly extended. In that spectrum, singletons are a convenient shortcut to avoid the decision entirely and just cram new features into an interface that's not flexible enough to meet all their requirements.

### Software Doesn't Exist in a Vacuum

Sometimes it's easy to prefer short-term convenience over long-term maintainability, but the truth is that most of the code we write has users (including other developers) whose sanity has to be considered as well. Let's say you're writing a library or API like [the one above](#singletons-are-a-temptation) that uses a singleton pattern, and you've managed to avoid the pitfalls we've covered so far. Is your design flexible enough to support any reasonable use-case? Probably not.

### The Weakest Link of Composition

Analogous to the common idiom, software is only as composable as its least composable component. Unfortunately, this means we can spend a whole lot of time focusing on composability, but as soon as we introduce one defect in the design, it will negatively impact everything that depends on it. This is especially true of the singleton pattern.

The following example using `for_each_with_policy` gets a bit convoluted, but it shows how confusing it is for users when a singleton causes composition to break down.

```cpp
#include <execution>
#include <span>
#include <string_view>

auto invoke_with_policy(execution_policy_t policy, auto func) {
  set_policy(policy);
  return func();
}

// This could call `set_policy` or `invoke_with_policy`
void do_other_thing();

int main(int argc, char *argv[]) {
  const auto args = std::span{argv, argv + argc};

  invoke_with_policy(std::execution::par, [=] {
    do_other_thing();

    // this probably intends to use `par`, but actually
    // depends on how `do_other_thing` is implemented
    for_each_with_policy(args, [](std::string_view arg) {
      // ...
    });
  });
}
```

It's tempting to blame the user who doesn't expect `do_other_thing` to change the execution policy, or the implementation of `invoke_with_policy` for not restoring the previous execution policy before returning (nice job if you already noticed that), but the issue underlying it all is that the singleton pattern makes it too easy to introduce both safety and logical errors.

I think the stage has now been set by demonstrating a few different ways that singletons can cause headaches, and also by determining some compelling reasons for their continued existence plaguing our codebases in spite of our best efforts. Now we're ready to start exploring another possible approach. Is there a way we can harness the convenience of singletons without their pitfalls?

---

## Automatic Storage Duration

To answer that question, we need to look at how **automatic storage duration** variables behave. In contrast with static storage duration, these lifetimes are well-defined and have a strict first-in-last-out ordering. Take the following program for example.

```cpp
#include <print>

struct printed {
  char id;

  printed(char name) : id(name) {
    std::println("construct {}", id);
  }

  ~printed() {
    std::println("destroy {}", id);
  }
};

auto f() {
  std::println("begin f");
  const auto y = printed{'y'};
  const auto z = printed{'z'};
  std::println("end f");
}

int main() {
  std::println("begin main");
  const auto a = printed{'a'};
  f();
  const auto b = printed{'b'};
  std::println("end main");
}
```

This produces the following output.

```txt
begin main
construct a
begin f
construct y
construct z
end f
destroy z
destroy y
construct b
end main
destroy b
destroy a
```

It's interesting to note how this ordering would be well-suited if `b`, `y` or `z` were to hypothetically have access to `a`, or if `z` were to have access to `y`. In general, no pair of lifetimes with automatic storage duration partially overlap. Either one lifetime spans over another entirely, or they do not intersect at all.

### Separating Concerns with Dependency Injection?

Revisiting the earlier topic, a common (mis)use of singletons is to facilitate a poor man's [dependency injection](https://stackoverflow.com/q/130794). But rather than injecting a dependency through a constructor parameter, it's exposed to the consumer as a singleton. While this frees the consumer from the responsibility of maintaining ownership, it does not successfully separate concerns like typical dependency injection does, because the consumer still needs explicit knowledge of the singleton in order to access it, and that really isn't a net improvement, it's just a different set of problems to deal with.

Note this is slightly different than the example with `invoke_with_policy`, because dependency injection typically deals with a *class* rather than an *algorithm* (although C++20 range adaptors and co-routines begin to blur the line here). The distinction is that an algorithm doesn't have data members to initialize, since it is not a container, but a class constructor must reason about either the ownership or lifetime of its dependencies in order to safely initialize its data members and avoid potential dangling references before its destruction.

### Delegation Requires Mixing Concerns

Consider the following software infrastructure.

{{< mermaid >}}
graph TD
A[Application] --> |calls| B[Business Logic]
B --> |calls| L[Library]
A --> |injects dependency| L
{{< /mermaid >}}

Nothing actually requires the business logic to know about the dependency injection, but because it is typically done through parameter passing, the business logic must declare a parameter to forward the dependency through to the library. This delegation through intermediate components in the call stack is an antipattern that React labels as ["prop drilling"](https://react.dev/learn/passing-data-deeply-with-context).

---

## A Contextual Approach

React provides the Context API as a solution for avoiding this "prop drilling" antipattern through intermediate components like `Navbar` and `Section` demonstrated in the following component tree.

```js
const theme = { linkColor: 'green' };
```

{{< mermaid >}}
graph TD
A["App theme={theme}"] --> B["Navbar theme={props.theme}"]
B --> C["Item color={props.theme.linkColor}"]
B --> D["Item color={props.theme.linkColor}"]
A --> E["Section theme={props.theme}"]
E --> F["Link color={props.theme.linkColor}"]
{{< /mermaid >}}

Instead, each component can directly consume its `color` provided by the context accessible within its scope.

```js
import { createContext, useContext } from 'react';

const ThemeContext = createContext(null);
const useTheme = () => useContext(ThemeContext);
const theme = { linkColor: 'green' };
```


{{< mermaid >}}
graph TD
A[App] --> P["ThemeContext value={theme}"]
P --> B[Navbar]
B --> C["Item color={useTheme().linkColor}"]
B --> D["Item color={useTheme().linkColor}"]
P --> E[Section]
E --> F["Link color={useTheme().linkColor}"]
{{< /mermaid >}}

Individual components can even consume different colors through the same context when multiple providers exist in the component tree:

```js
const navbarTheme = { linkColor: 'blue' };
```

{{< mermaid >}}
graph TD
A[App] --> P["ThemeContext value={theme}"]
P --> B[Navbar]
B --> Q["ThemeContext value={navbarTheme}"]
Q --> C["Item color={useTheme().linkColor}"]
Q --> D["Item color={useTheme().linkColor}"]
P --> E[Section]
E --> F["Link color={useTheme().linkColor}"]
{{< /mermaid >}}

This highlights a couple key points:

- Contexts are not singletons.
- Consumers use the most local provider in scope.

One additional point that isn't demonstrated in this example is that consumers can safely detect when no provider is in scope, by obtaining the default value (in this case `null`) when using the context.

Consider how this component tree compares with the call stack, where each component lifetime behaves like a C++ automatic storage duration variable. 

### Designing a Less Error-Prone Pattern

Let's jump right into how this pattern might be implemented in C++.

```cpp
#include <memory>
#include <utility>

// this is how we provide scoped objects
template <class T>
thread_local T *context = nullptr;

template <class T>
class provider {
  // current object with automatic storage duration
  T inner;
  // pointer to previous object
  T *outer;

public:
  provider(T value)
      : inner(value),
        // exchange previous scope with current scope
        outer(std::exchange(
            context<T>,
            std::addressof(inner))) {}

  provider(const provider &) = delete;
  provider(provider &&) = delete;

  // restore previous scope on destruction
  ~provider() { context<T> = outer; }
};

// provide an object that can be accessed with get_context<T>()
template <class T>
auto make_context(T value) { return provider<T>(value); }

// access the object provided within the current scope
template <class T>
T *get_context() { return context<T>; }
```

Here's how we can apply this to our previous unit testing example.

(https://godbolt.org/z/43xc76Y9K)

```cpp
#include <functional>
#include <optional>

#include "gtest/gtest.h"

auto set_enabled(bool state) {
  return make_context<bool>(state);
}

bool get_enabled() { return *get_context<bool>(); }

// same as before
void dispatch(int event, std::function<void(int)> callback);

// set default to disabled with static storage duration
auto disabled_ctx = set_enabled(false);

TEST(Dispatcher, DropsByDefault) {
  auto actual = std::optional<int>{};
  const auto expected = std::nullopt;

  ASSERT_FALSE(get_enabled());
  dispatch(99, [&](int result) { actual = result; });

  EXPECT_EQ(actual, expected);
}

TEST(Dispatcher, ForwardsWhenEnabled) {
  auto actual = std::optional<int>{};
  const auto expected = 42;

  // set scope to enabled with automatic storage duration
  auto enabled_ctx = set_enabled(true);
  dispatch(expected, [&](int result) { actual = result; });

  EXPECT_EQ(actual, expected);
}
```

And finally, what about our example with execution policy?

```cpp
#include <algorithm>
#include <concepts>
#include <execution>
#include <ranges>
#include <variant>

using execution_policy_t = std::variant<
    std::execution::sequenced_policy,
    std::execution::parallel_policy,
    std::execution::parallel_unsequenced_policy,
    std::execution::unsequenced_policy>;

auto set_policy(execution_policy_t policy) {
  return make_context<execution_policy_t>(policy);
}

execution_policy_t get_policy() {
  return *get_context<execution_policy_t>();
}

// same as before
void for_each_with_policy(
    std::ranges::common_range auto &&range,
    auto func);

auto invoke_with_policy(execution_policy_t policy, auto func) {
  // set scope to `policy` with automatic storage duration
  auto policy_ctx = set_policy(policy);
  return func();
}

// This could call `set_policy` or `invoke_with_policy`
void do_other_thing();

int main(int argc, char *argv[]) {
  const auto args = std::span{argv, argv + argc};

  invoke_with_policy(std::execution::par, [=] {
    do_other_thing();

    // this will now use `par` no matter how `do_other_thing`
    // is implemented
    for_each_with_policy(args, [](std::string_view arg) {
      // ...
    });
  });
}
```

A more refined implementation of these utility functions `make_context<T>(...)` and `get_context<T>()` that allow in-place construction, more flexible reference semantics, const-qualification, and noexcept-correctness is available at https://github.com/patrickroberts/pr along with supporting documentation.

In conclusion, I think the key takeaways should be this.

- Singletons are an error-prone shortcut to circumvent complex design choices.
- The usage of antipatterns doesn't originate from malicious intent, but rather from a lack of understanding, experience, or time to be able to design a proper solution.
- Inspiration for solutions can be found by studying the patterns used within other successful languages and libraries.
