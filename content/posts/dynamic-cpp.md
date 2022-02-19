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
- Add expected console output for examples.

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

What if we need to decorate this mess? Welp, it's time to go cry in the corner. It can't be done reliably as there are a couple of problems:

- You will need to wrap and forward arguments for ALL the methods, not only pure-virtual ones. This leads to a ton of repetetive code that just asks to be messed up. Ask me how I know.
- The interface can't have private or protected virtual methods as these can be overriden but can't be called on an instance from outside the interface class. If they are present there is no clear and good solution. You just need to that the classes you're wrapping do not override them.
- You can only hope that the interface won't change. Removing or adding pure virtual methods isn't actually a problem(to an extent, it does break a build). New default-implemented methods are truly dangerous though as they will lead to all kinds of hard-to-detect problems. Again - ask me how I know.

### Non-Solution 1

Is there a better way? Don't create such things and don't work with libraries that require you to use them ;). If you control the interface, make it use a Non-Virtual Interface idiom. It would look something like this.

```C++

class Interface
{
public:
    void PublicMethod()
    {
        // Some pre-conditions
        VirtualMethod();
        // Some post-conditions
    }

protected:
    virtual void VirtualMethod() = 0;
};

```

This way you don't need to use a separate decorator class at all.

### Non-Solution 2

What if you don't control the interface(a common thing) but you know about all the possible implementations and have access to some form of RTTI? If we don't consider a possibility of the implementations being marked as `final` we can quite easily write something as follows:

```C++

template<typename T>
class Decorator : public T
{
    // Putting a static_assert or an enable_if or a concept here so we don't mess up too hard due to duck-typing in the template.
    static_assert(std::is_base_of_v<Interface, T>, "We're only meant to be decorating a certain interface.");

    // T needs to have a copy constructor for this to be generic...
    Decorator(const T& t)
        : T(t) {}

    // ...or at least a move constructor
    Decorator(T&& t)
        : T(std::move(t)) {}

    void TheOnlyMethodWeOverride() override
    {
        fmt::print("Decoration is love. Decoration is life. \n");
        T::TheOnlyMethodWeOverride();
    }
};

std::unique_ptr<Interface> Decorate(std::unique_ptr<Interface> original)
{

    if (auto concrete = dynamic_cast<Concrete1*>(original.get()))
    {
        return std::make_unique<Decorator<Concrete1>>(std::move(*concrete));
    }
    else if (auto concrete = dynamic_cast<Concrete2*>(original.get()))
    {
        return std::make_unique<Decorator<Concrete2>>(std::move(*concrete));
    }
    else
    {
        return {nullptr};
    }
}

```

In the example above for the sake of simplicity I assumed that we don't use `-fno-rtti` so we can use `dynamic_cast`. If we in fact do have `-fno-rtti` enabled it's often the case that there is a homegrown RTTI substitution in place be it Unreal-style huge framework framework or just a `GetType` returning an enum or a string. Also I did the most simple and stupid thing - just an `if-else` but just a few (dozen) additional lines of code you can make a type list and iterate over it. This is not a perfect solution but it is certainly not as attrociuos as the things we gonna look at a bit further and I saw similar things in production quite a lot.

### Non-Solution 3

But what if really do need to decorate a naughty interface and you really can't know your implementations beforehand? What should you do? Except reconsidering your life choices ofcourse.

Let's think. What would be a clean solution? If we knew the class of the object we were decorating we could've just inherit a new class from it and define the methods we need(as we did in the previous one). But as we don't know the implementation set beforehand - say due to it user-extensible via a dll plugin system or just being in the actively-changing chunk of codebase we don't control this is not an option. See, even though inheritance is the primary mean to do dynamic polymorphism in C++(at least until very recently) it itself is inherently a static thing. 

What if it wasn't? Let's see what you can do in more dynamic languages like Python or Ruby where you can easily modify type data in-flight. We can almost literally repeat our previous implementation but due to types being fully accessible in runtime we don't have to rely on knowing all the possible implementations.

```Python

class Interface:
    def method(self):
        pass

class Impl(Interface):
    def __init__(self):
        self.data = "My precious data"
    def method1(self):
        print("Impl method1 " + self.data)
    def method2(self):
        print("Impl method2")

obj = Impl()
obj.method1()
obj.method2()

print()

def decorate(obj):
    old_class = type(obj)
    class Decorated(old_class):
        def __init__(self):
            self.__dict__.update(obj.__dict__)
        
        def method1(self):
            print("Decoration!")
            super(type(self), self).method1()
    return Decorated()

obj = decorate(obj)
obj.method1()
obj.method2()

```

Neat! (I know that an actual Python programmer probably wouldn't be happy to see this in prod, yes)
We can't have this in C++ though. Or can we? Let's look at another way this can be done in Python.

```Python

from types import MethodType

class Interface:
    def method(self):
        pass

class Impl1(Interface):
    def __init__(self):
        self.data = "My precious data"

    def method(self):
        print("Impl1 " + self.data)
    
obj = Impl1()
obj.method()

print()

def decorate(obj):
    old_method = obj.method
    def decorated_method(self):
        print("Decorated!!!")
        old_method.__func__(self)
    obj.method = MethodType(decorated_method, obj)
    return obj

obj = decorate(obj)
obj.method()

```

Here we just straight up monkey-patch a method on an object. What does it have to do with C++ you could ask? Well you can kinda do this here too!

## Virtual function tables