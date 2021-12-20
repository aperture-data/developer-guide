At Aperture, we mostly follow the [Google Style Guide (GSG)](https://google.github.io/styleguide/cppguide.html), but we make a significant departure in that we use C++ exceptions for error handling and [they do not](https://google.github.io/styleguide/cppguide.html#Exceptions). Google’s prohibition of exceptions is motivated primarily by legacy compatibility concerns that we do not share, and there is good reason for us to use them. However, safe exception handling requires additional conventions not covered in the GSG, including RAII. This document outlines some supplemental stylistic guidelines to promote robust, consistent, and maintainable code in the presence of exception-based error handling.


# RAII Guidelines

[Resource Allocation Is Initialization (RAII)](https://en.cppreference.com/w/cpp/language/raii) is a design pattern in C++ where allocated resources are tied directly to the lifetime of an object. Using RAII, any resource that needs to be acquired and released is wrapped in a class such that it is acquired in the constructor and released in the destructor. Examples of RAII classes include [`std::unique_ptr<>`](https://en.cppreference.com/w/cpp/memory/unique_ptr), [`std::lock_guard<>`](https://en.cppreference.com/w/cpp/thread/lock_guard), and [`std::ifstream`](https://en.cppreference.com/w/cpp/io/basic_ifstream).

RAII has several benefits. First and foremost, it guarantees that resources are released. This simplifies garbage collection and prevents leaks. This is particularly important in the presence of exceptions because it is practically impossible to guarantee that resources are released during stack unwinding without RAII.

Additionally, this simplifies development by keeping resource acquisition and release adjacent in source and by eliminating transient invalid states (eg. an object has been created, but its resources haven’t been allocated yet).

The following are practical guidelines for developing RAII-compliant code.

## 1. Prefer smart pointers to `new`, `delete`

This [prevents leaks](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Re-raii). It also provides better context about how objects are managed compared to raw pointers.

### Exceptions

- Use of `new` is ok when constructing or assigning to a `unique_ptr` (until C++14 where [std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique) is available).

## 2. Wrap pairwise system & library calls

Eg. [`SSL_CTX_new()`](https://www.openssl.org/docs/manmaster/man3/SSL_CTX_new.html) and [`SSL_CTX_free()`](https://www.openssl.org/docs/manmaster/man3/SSL_CTX_free.html) are wrapped in [`OpenSSLPointer<SSL_CTX>`](https://github.com/aperture-data/aperturedb-cpp/blob/0421139ebcb03997e02864d9c92ae8f6af8a01b8/src/comm/TLS.h#L27-L42). Interfaces should be expressed in terms of the RAII wrapper type rather than the raw type whenever possible to enforce its usage.

## 3. Exceptions should not escape a destructor

Even though compilers don’t consistently enforce it, [destructors are implicitly `noexcept`](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-dtor-noexcept). If a destructor throws during exception handling, the process will immediately terminate (which is probably not what we want, even if the issue is fatal). Any calls made by a destructor should be themselvesnoexcept or wrapped in a `try..catch`.

## 4. Initialize values once

Member values should be initialized in the constructor with minimal modifications. All members should be initialized as a postcondition of the constructor. If that is not possible, the constructor should throw.

```cpp
// bad
class MyObject
{
    int _a;
    std::string _b;
    Thing* _c;
public:
    MyObject() {}
    void SetA(int a) { _a = a; }
    void SetB(const std::string& b) { _b = b; }
    void SetC(Arg arg) { _c = new Thing(arg); }
    ~MyObject() { if(_c)delete _c; }
};
```

`MyObject` constructor creates a valid but uninitialized object. This places the onus on the caller to set member values, which makes calling code verbose and error prone. It is expensive for a developer to use because it requires understanding of the internal details of `MyObject`. Also, care must be taken not to leak `_c` (eg. not calling `SetC()` multiple times).

```cpp
// better
class MyObject
{
    int _a;
    std::string _b;
    std::unique_ptr<Thing> _c;
public:
    MyObject(int a, const std::string& b, Arg arg) {
        _a = a;
        _b = b;
        _c = new Thing(arg);
    }
};
```

This interface informs how `MyObject` should be constructed, and guarantees that all members are initialized in the constructor. If the call to `new Thing(arg)` throws, the `MyObject` instance is not created and nothing is leaked.

```cpp
// best
class MyObject
{
    const int _a;
    const std::string _b;
    Thing _c;
public:
    MyObject(int a, std::string b, Arg arg)
        : _a(a)
        , _b(std::move(b))
        , _c(arg)
    {}
};
```

Members are initialized in place instead of being default-initialized and reset in the ctor body. `_c` is constructed in place as a native member, removing the need for the `unique_ptr` indirection. `_b` is move-constructed, which saves a string copy when the constructor is called with an r-value for `b`.

## 5. Leverage compiler for construction/destruction ordering

The compiler guarantees that members are constructed in declared order and destructed in the inverse order. Take advantage of this to control the lifecycles of related/interdependent objects.

```cpp
// good
class Server
{
    Config _cfg;
    Logger _log;
    APIHandler _handler;
    std::unique_ptr<Thread>_worker;

    void do_work(APIHandler& handler);

public:
    Server(const std::string& config_file)
    : _cfg(config_file)
    , _log(_cfg)
    , _handler(_cfg, _log)
    , _worker()
    {
        _worker.reset(new Thread([this](){
            do_work(_handler);
        }));
    }
};
```

Each member depends on those before it. When each member is constructed, this implementation guarantees that all of its dependencies are already present. The implicit `~Server()` destructs members in the reverse order that they were constructed, guaranteeing that no member will outlive its dependencies.

## 6. Prefer explicit lifecycle management & reference passing over statics/globals

If a static object requires explicit initialization and/or destruction, consider using a non-static RAII object instead.

Consider the following global singleton pattern.

```cpp
class GlobalObject
{
    GlobalObject();
    static std::unique_ptr<GlobalObject> _inst;
public:
    static bool init() {if(!_inst) _inst = new GlobalObject(); }
    static void destroy() { _inst.reset(); }
    static GlobalObject& instance() { init(); return *_inst; }
};
```

This offers some conveniences such as lazy initialization, universal availability, and a uniqueness guarantee.

But there is no way to explicitly control its lifecycle. When there are multiple objects using this pattern, initialization & destruction ordering becomes geometrically complicated, opaque, and error prone.

Once we start linking shared libraries, even the uniqueness guarantee [breaks down in subtle and unintuitive ways](https://techunravel.blogspot.com/2011/10/staticglobal-variable-and-shared.html).

By contrast, non-static objects produce simpler code, fewer edge cases, and are easier to reason about. Additionally, compared with a static `instance()` accessor, passing references to an object makes it easier to keep track of how and where that object is used, and helps control spaghettification of the code.

Note also that a static global object can only reliably depend on other static globals, so they are incompatible with compiler-oriented initialization & destruction ordering as described [above](#5-leverage-compiler-for-constructiondestruction-ordering).

### Exceptions

- Constant immutable data does not need an explicit lifecycle. In such cases, prefer declaring immutable data `constexpr`.
- There are rare cases when the lifecycle of an object cannot be known at compile time. An example is a resource that is shared between threads, and it is indeterminate which thread will be the last to use it. In such cases ([and only those cases](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rf-shared_ptr)), consider using [`std::shared_ptr<>`](https://en.cppreference.com/w/cpp/memory/shared_ptr).

## 7. Minimize work performed in destructors

When there are multiple ways to release a resource, prefer the simplest option by default. In the following contrived example, we have an RAII transaction object with two options for closing the transaction.

```cpp
struct trans_t;
void open_transaction(trans_t&);
void abort_transaction(trans_t&); // light operation
void commit_transaction(trans_t&); // heavy operation

class Transaction
{
    trans_t _tx;
    bool _open;
public:
    Transaction()
    :_open(false) {
        open_transaction(_tx);
        _open = true;
    }

    void Commit() {
        if(_open) {
            commit_transaction(_tx);
            _open = false;
        }
    
    }
    void Abort() {
        if(_open) {
            abort_transaction(_tx);
            _open = false;
        }
    }

    // abort by default
    ~Transaction() {
        try {
            Abort();
        } catch(...) {}
    }
};
```

By convention, the destructor should call `Abort()` because it is the lighter of the two, and it becomes the responsibility of the caller to `Commit()` the transaction explicitly when desired.

  
