Actual storage is placed in the static `unsigned char *` pointer called `SideTableBuf` with proper size capable to fit `StripedMap<SideTable>`.

```c++
alignas(StripedMap<SideTable>) static uint8_t SideTableBuf[sizeof(StripedMap<SideTable>)];
```

It's not good solution, but it works. Thanks to Apple engineers for putting comments of reasons.

> // We cannot use a C++ static initializer to initialize SideTables because <br>
> // libc calls us before our C++ initializers run. We also don't want a global <br>
> // pointer to this struct because of the extra indirection. <br>
> // Do it the hard way.

Initialization is performed using simple `static` function. 

```c++
static void SideTableInit() {
    new (SideTableBuf) StripedMap<SideTable>();
}
```

I'm not C++ man, so it was a little bit tricky for me to understand. `new` operator have a lot of variations. 
Current call is similar to so-called *placement new* operator. 

```c++
void* operator new(std::size_t, void*)
```

This version takes additional pointer parameter, which is used to refer to pre-allocated storage. 
So C++ class constructor uses this storage without allocating any other. The other effect, that the result of `new` isn't used. 
It has the same reason, because *placement new* operator use pre-allocated storage the value (void *pointer) will be the same as parameter.

And in order to use this storage, `uint8_t *` pointer is casted to actual stored class:

```c++
static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```

What's about `StripedMap`?
```c++
template<typename T>
class StripedMap {
    enum { CacheLineSize = 64 };
// ..
    enum { StripeCount = 64 };
// ..
    struct PaddedT {
        T value alignas(CacheLineSize);
    };

    PaddedT array[StripeCount];

    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }
    
 public:
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }
// ...
};
```

`StripedMap` is the implementation of a dictionary (map) class, where `void *` pointer is a key, and `T` is a template value. Usually, programmers deal with a hash-map. Hash-map has two valuable specifics: 

- this kind of map depends heavily on the quality of hash-function
- there is a mechanism for conflict's solving, when two keys have the same hash. One of the ways is to put store linked list of key-values, instead of value.

`StripedMap` has something like a hash-function, it is the following expression:

```c++
((addr >> 4) ^ (addr >> 9)) % StripeCount
```

Right shift (`>>`) and xor (`^`) operators can be executed as a few instructions on the most of the modern CPUs, and the modulus operator (`%`) also rather trivial. It means that this expression is calculated pretty fast. To get some overview of the statistical distribution, I used simplest simulation of calculation hash using memory pointers by allocating small chunks of memory in for-loop without releasing them (it will guarantee that adress will be unique). Variable parameters were pointer values count (amount of iterations), different chunks for allocations. In all cases distribution was close to the discrete uniform distribution (equal or almost equal amount of matches on all positions). Consequently, we can think about this function as a hash function.

Second point is the most interesting. As I said previously, hash-maps usually contains mechanism for handling conflicts during filling value for key and keeping data structure able to return value for key later. `StripedMap` is a different thing. It is an internal storage implemented as an statically-defined array. That means that internal storage will be allocated and filled at the end of StripeMap creation. Also it's clear that this is read-only data structure, developer can only get value by key using overloaded operators in `public:` class interface section. So if we collect these facts and keep in mind word `striping`, it's become clear what this class actually do. It splits access to certain recources based on the pointer. Resources are counted as equal, so the main goal to have stable repeatable access to the same resources each time (no matter what exactly this resource will be). 

> // StripedMap<T> is a map of void* -> T, sized appropriately <br/>
> // for cache-friendly lock striping. <br/>
> // For example, this may be used as StripedMap<spinlock_t> <br/>
> // or as StripedMap<SomeStruct> where SomeStruct stores a spin lock.

Comment also points out about cache-friendlyness

**References:**

- [Reference page for `new`](http://en.cppreference.com/w/cpp/language/new)