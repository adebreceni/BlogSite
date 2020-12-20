# Type erasure

So you wake up one morning simply just aching for duck-typing in your favorite programming language of choice. Excited as an eight-years-old on Christmas morning, you jump out of bed and run downstairs in your pajamas (because you surely don't sleep with your laptop by your bed). As you hop down in your Herman Miller chair, pull forward your mechanical keyboard, and wiggle your custom made ergonomic mouse so your computer wakes up as well, only then do you realize that, "oh yeah, my favorite language of choice is `c++`". Worry you not, put on a coffee, clench your butt cheeks and lets get started.

In the following we will assume that you care about nothing more than to pass around *all* objects that support the `std::string serialize() const` method, and, of course, to eventually call this method.

### Template it away

Of course as a seasoned `c++` programmer, your knee jerk reaction is to put on your template-writing-hat and template away. 

```scala
template<class T>
void g(const T& any) {
  std::cout << any.serialize();
}

template<class T>
void f(const T& any) {
  g(any);
}
```

Now, if you are really that seasoned, you are obviously using `c++35` so you are more than happy to create the `Serializable` concept to enforce the existence of the method at every step of the way. If you are not using concepts, you will get the ever-so-helpful error messages, indicating at the site of the actual method access, that you, in fact, did not pass on an object with serializable method. (yeah, yeah, you could also sprinkle `enable_if`s (and `static_assert`s) all around even before `c++35`). 

### Erase the hell out of that type

You know how `std::function` works, don't you? 

A quick recap: 

There is a *base* class encapsulating the callable object with all kinds of pure virtual methods like `__clone`, `operator()`.

Then there is a pointer inside `std::function` pointing to a *base* class instance (which may or may not reside inside the fixed size buffer for small-object optimization). 

Then whenever you construct a new `std::function` or just assign to it, it inspects the actual type of the callable and instantiates a new class inheriting from *base,* implementing all those virtual methods. 

With this knowledge we are now ready to cook up our own implementation.

```scala
class Serializable {
  // the base class with pure virtual methods
  struct Base {
    Base(void* instance) : instance_{instance} {}
    virtual std::string serialize() const = 0;
    void* instance_;
  };

  // the derived class that knows about the actual type
  // so can implement the virtual methods
  template<typename T>
  struct Derived : Base {
    using Base::Base;
    std::string serialize() const override {
      return static_cast<const T*>(instance_)->serialize();
    }
  };
 public:
  // construct the derived object encapsulating any serializable
  // into our happy little buffer
  template<typename T, typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Serializable>>>
  Serializable(T&& value) {
    new (&buffer_) Derived<std::decay_t<T>> (static_cast<void*>(&value));
  }

  template<typename T, typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Serializable>>>
  Serializable& operator=(T&& value) {
    new (&buffer_) Derived<std::decay_t<T>> (static_cast<void*>(&value));
    return *this;
  }

  std::string serialize() const {
    return base_->serialize();
  }
  
 private:
  alignas(alignof(Base)) char buffer_[sizeof(Base)];
  Base* const base_{reinterpret_cast<Base*>(&buffer_)};
};
```

Cute. Let's see it in use.

```scala
struct A {
  std::string serialize() const  {
    return "Hello from \"A\"";
  }
};

struct B {
  std::string serialize() const  {
    return "Hello from \"B\"";
  }
};

void f(Serializable x) {
  std::cout << x.serialize() << std::endl;
}

int main() {
  A a{};
  B b{};

  f(a);
  f(b);
}
```

Now, I can hear you grinding your teeth, stop that please, it is really annoying. 

What kind of overhead are we working with here? A quick glance at the `sizeof(Serializable)` shows us whopping `24` bytes. Of course we all know that copying bytes (especially `24`!) is the leading cause of death in Andorra, so we desperately would like to decrease that. 

### Plan B

We need access to the object, and we need access to the method. There is simply no going around that.

```scala
class Serializable {
  template<typename T>
  struct MethodAccessor {
    static std::string serialize(void* instance) {
      return static_cast<const T*>(instance)->serialize();
    }
  };
 public:
  template<typename T, typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Serializable>>>
  Serializable(T&& value) {
    instance_ = static_cast<void*>(&value);
    method_ = MethodAccessor<std::decay_t<T>>::serialize;
  }

  template<typename T, typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Serializable>>>
  Serializable& operator=(T&& value) {
    instance_ = static_cast<void*>(&value);
    method_ = MethodAccessor<std::decay_t<T>>::serialize;
    return *this;
  }

  std::string serialize() const {
    return method_(instance_);
  }
  
 private:
  void* instance_;
  std::string (*method_)(void*);
};
```

Look at that `MethodAccessor` beast. If we ask the compiler nicely it will generate us any contraption we wish. In this case, whatever type we throw at the constructor, a brand new function is created, capable of calling said type's `serialize` method. (Obviously you can make do with a template function just as well)

Is this complete? No.

Is this implementation safe? That is a definite no.

Should I use this specific code in production? Please don't.

As an exercise go ahead and create a `std::function` variant that can wrap non-copiable callables.