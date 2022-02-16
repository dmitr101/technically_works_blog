+++
title = "Making C++ more dynamic"
date = 2022-02-16T21:45:00+01:00
images = []
tags = []
categories = []
draft = true
+++

# TODO

- Should I bring diagram. 
- Should I joke about bringing diagram. 
- Make big samples collapsable.
- Make an example project with examples?
- C++ code style. Right now it's Unreal and kinda ugly for my taste.
- Example style. Abstract or concrete? Should maybe do a logger or config provider or something like that.(Probably not socket subsytem, lol)
- Examples godbolt links where applicable.

# TODOEND


## Little story
Let's talk about decorators. Yeah that thing from the GoF book(or the hardest thing to wrap your head around if you're anything like me and was learning Python at some point in your life).
The basic premise of a decorator is pretty simple: take a thing and add some behaviour to it while keeping the interface. In simple C++ it could look something like this ([Godbolt link](https://godbolt.org/z/x7sc676qj)):

```C++
#include <fmt/core.h>
#include <memory>

struct Interface
{
    virtual void Method() = 0;
};

struct SomeImplementation : Interface
{
    void Method() override
    {
        fmt::print("SomeImplementation! \n");
    }
};

struct Decorator : Interface
{
    Decorator(std::unique_ptr<Interface> Inner)
        : Inner{std::move(Inner)}
    {}

    void Method() override
    {
        fmt::print("Decorated!");
        Inner->Method();
    }

private:
    std::unique_ptr<Interface> Inner;
};

void UseInterface(Interface& interface)
{
    interface.Method();
}

int main()
{
    auto impl = std::make_unique<SomeImplementation>();
    UseInterface(*impl);
    auto decorated = std::make_unique<Decorator>(std::move(impl));
    UseInterface(*decorated);
}
```

And in cases like this one everything usually works pretty nicely. Sure, we need dynamic allocations and (arguably) a couple of smart pointers here and there but that's just the way dynamic polymorphism is in C++ if we are talking open type sets(which are required in many cases) and not `variant` everywhere. This thing shouldn't be anywhere near the hot path but for general plumbing around the codebase it can be a great tool. Especially if you don't own chunks of the codebase and they talk to each other via interfaces.

### There is a little problem...

No it's not that that OOP is dead and we all should switch to FP/pure C/something else. Even though it very well might that too. Anyway.

In large codebases interfaces as simple as the one from our example can be pretty hard to come by. Usually it looks a bit more like this.

```C++
class Interface
{
public:
    // A base set of pure virtual methods. They probably live here from the very start.
    virtual void Method1() = 0;
    virtual void Method2() = 0;

    // A couple of optional methods with empty base implementations
    virtual void Method3() {}
    virtual void Method4() {}

    // A couple of methods with implementations still marked as virtual
    virtual void Method5()
    {
        Method1();
        Method4();
        fmt::print("Some default stuff");
    }

    virtual void Method6()
    {
        Method5();
        Method3();
        fmt::print("Some other default stuff");
    }

    // And a protected virtual method just to add some spice
protected:
    virtual Method7()
    {
        fmt::print("I'm protected so you can't call me on anything besides your own instance!");
    }
};
```

What if we need to decorate this mess? Welp, it's time to go cry in the corner. You can't. Thanks for reading, see you later.