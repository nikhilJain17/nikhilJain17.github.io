+++
date = '2026-07-30T15:35:25-07:00'
draft = false 
title = 'Type erasure in C++'
summary = 'How to store parameterized types in a C++ container'
tags = ['c++', 'type-systems']
+++

## The problem
I had an interview question where I had to create a library to execute pipelines made of up stages. To keep things simple, let's only consider the case where a stage takes in a single input and outputs a single output. These inputs and outputs could be of any type.

I templated the stage class with input and output types to look something like this (with some details removed):

```C++
template <class In, class Out>
class Stage {
    std::function<Out(In)> func; 
    Out calculate(In arg) {
        // return func(arg)
    }

    StageKey get_key() {
        return key;
    }
};
```

Then, I wanted to store the stages in a map like:

```C++
std::unordered_map<StageKey, Stage> pipeline_stages;
```
C++ maps are not like Python dicts, in that all keys and all values must be of the same type declared at compile time. This is since the elements in a container must be of the same size.

But `Stage` is a parameterized type! So a `Stage<Token, ProcessedToken>` is a different type than a `Stage<Transaction, Balance>`, and both cannot be stored in the same container, since they are of different sizes! 

Then how do we store `Stage`s in our map?

## The solution
The solution is type erasure. We want to unify the different `Stage<In, Out>` values under a single interface that has no concept of the type parameters `In` and `Out`, and then somehow store this base type in a container while being able to fetch the parameterized type information when desired. How do we do this?

This is done in two steps. First, we must unify stages under a common interface that has no reference to `In` or `Out` types.

### 1. A type erased interface

We declare an abstract `IStage` class that defines an interface for `Stage`s, but with no type information. 

```C++
class IStage {
public:
    virtual ~IStage() = default;
    virtual StageKey get_key() = 0;
    virtual void calculate(Context &context) = 0;
};
```

We have to modify `calculate()` to no longer return an `Out` and no longer take an `In`. Otherwise this wouldn't be type erased and we would still have the same problem!

So instead, we refactor `calculate()` to mutate an external `Context` which stores inputs and outputs of stages. Each `Stage<In,Out>::calculate` gets its own body containing its own `any_cast<const In&>` and its own call through `func`. The types move out of the signature and into the body. This way, our `IStage` can be free of types.

Then, we can implement Stage as a subclass of our interface class.
```C++
template <class In, class Out>
class Stage : public IStage {
    std::function<Out(In)> func; 
public:
    void calculate(Context &context) override {
        // read input from Context
        // calculate func
        // write output to Context
    }

    StageKey get_key() override {
        return key;
    }
};
```

{{< aside title="An aside" >}}
You might be wondering...aren't we just pushing the problem onto the `Context` object? Partially. We can resolve this by defining `Context` like
```C++
class Context {
    // map StageKey --> Output value
    std::unordered_map<StageKey, std::any> stage_results;
};
```
and when we are reading the output of a previous stage as an input into our stage, we can cast it into `In` like 
```C++
try {
    // A stage may not mutate the input within Context, since other stages may read the same input.
    input =
        std::any_cast<const In &>(context.stage_results.at(dep));
    } catch (const std::bad_any_cast &) {
        // Should not happen, since type checking between `Port<Out>` and `func = std::function<Out(In)>` is done via Ports within add_stages
        throw Error::TypeMismatch;
    }
```

I also added a `Port<Out>` type as a handle for each stage to enforce type safety when creating a stage. The `Out` parameter is checked against the `func`'s type signature at compile time. But this is a tangent. 
Let's get back to the original problem of storing parameterized types in a single container.
{{< /aside >}}


### 2. Store pointers to concrete types
Now that we have a type-erased base class `IStage`, we can store stages in our map like 
```C++
std::unordered_map<StageKey, IStage> pipeline_stages;
```

But we can't access any of our concrete method implementations, since our values are of an abstract class type! Even if we instantiate a `Stage<In, Out>` and store it as an `IStage` in the map, all the data associated with `Stage<In, Out>` has been sliced off when it gets stored in the map. This happens since a container needs to store elements of the same size. So then how do we call the actual implementations we want?

The answer is to store instances of the concrete `Stage<In, Out>` derived types on the heap, and store upcasted pointers of type `IStage` in the map.

```C++
std::unordered_map<StageKey, std::unique_ptr<IStage>> pipeline_stages;
// takes type parameters on the way in, but erases types so downstream callers can just operate on `IStages`.
template <class In, class Out>
void add_stage() {
    std::unique_ptr<IStage> type_erased_stage_ptr = std::make_unique<Stage<In, Out>>();
    pipeline_stages.emplace(stage_key, std::move(type_erased_stage_ptr));
    // ...
}
```

Now, our `pipeline_stages` map stores type-erased pointers of type `IStage` that point to instances of the concrete derived class of type `Stage<In, Out>` on the heap! 

![A map of StageKey to unique_ptr<IStage>, where each pointer refers to a concrete Stage<In, Out> instance on the heap](/images/type_erasure_diagram.jpg)

So when we call the interface methods on the `IStage` pointers, we are actually dynamically dispatching to the implementations defined in the `Stage<In, Out>` class!

```C++
std::unordered_map<StageKey, std::unique_ptr<IStage>> pipeline_stages;
const auto& type_erased_ptr = pipeline_stages.at(stage_key);
type_erased_ptr->calculate(ctx); // calls Stage<In, Out>::calculate for In, Out on the value that was placed on the heap during insertion! 
```

So we have created a container that can store values of a parameterized type. Nice!

### Other applications
Type erasure is useful beyond enabling heterogeneous containers.

It can help remove templated code. Functions that operate on stages that are not concerned about the type parameters of the stage can simply operate on `IStage`.

```C++
template <class In, class Out>
void validate_stage(const Stage<In, Out> &stage);
```

vs

```C++
void validate_stage(const IStage &stage);
```

The template parameters on the first code snippet would end up infecting the entire codebase! Not only are these templates tedious, but they are also unnecessary in many cases where functions do not need to know about the details of a parameterized type and just need to interact with an abstract version of such a type.

We can also use it to store unnameable types, such as lambda expressions which must be declared as `auto` to deduce the unnameable type:
```C++
auto f = [x](int i) { return i + x; };  // type is something like __lambda_7a3f_impl
```

In fact, this is how `std::function` is implemented to capture lambdas! We can't just use `std::function` for the whole pipeline though, because it has type information in its signature. An `std::function<Out(In)>` still contains `In` and `Out` and needs to be type erased to store in a container.

We can also use this in cases where types are known at runtime but not compile time. 

### Downsides
We are trading compile-time type checking for runtime checking, since when we operate on `IStages`, the compiler doesn't know about `In` and `Out` and therefore cannot do any checking on their usages. 

We also call functions indirectly which incurs some vtable lookup overhead. Also, we cannot inline these functions and lose associated compiler optimizations, such as constant propagation. Additionally, each stage has to be allocated on the heap. 

### Conclusion
We saw how we can use type erasure to store heterogeneous types in the same container. We also saw how we can abstract away type parameters in a base type, and use dynamic dispatch to reach the concrete type's implementation.

### AI Disclosure
Code and blog post written by me, both lightly edited with Claude for correctness.