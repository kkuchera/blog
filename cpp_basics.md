# C++ Basics

There seems to be recurring misuse of some basic C++ concepts that I thought were worth summarizing. They are not presented in the order they occur most frequently, but rather in a topological order i.e. later points might depend on understanding earlier points.

## 1. Resource Acquisition Is Initialization
C++ has deterministic destructors. It is precisely known when an object's destructor is called, namely when the object goes out of scope. By acquiring resources during an object's initialization, and releasing them at destruction, resources are tied to the lifetime of the object. This is called *Resource Acquisition Is Initialization* or *RAII*. When the object goes out of scope, its resources are automatically released thus providing strong resource safety. A resource is anything that needs to be acquired and released such as a thread, lock, memory, file descriptor, etc.

Take the following code for example:
```cpp
void f(int i) {
    X* x = new X;
    if (i < x->value) return;
    x->do_something();
    delete x;
}
```

If `i < x->value` is `true`, or `x->do_something()` throws an exception, `x` is not deleted and the `f` leaks memory. This could be solved using a simple RAII wrapper `Y`:
```cpp
struct Y {
    X* x;
    Y() { x = new X;}
    ~Y() { delete x; }
};
```

Using `Y`, `f` becomes
```cpp
void f(int i) {
    Y y = Y();
    if (i < y.x->value) return;
    y.x->do_something();
}
```

When `y` goes out of scope, whether it's because of a premature return or an exception, the resource `x` is properly released. RAII eliminates `new` operations in general code and keeps them buried inside well-behaved abstractions. Avoiding naked `new` and `delete` makes code less error prone and easier to keep free from memory leaks.

The same technique is used to implement a simple smart pointer
```cpp
template <typename T>
struct smart_ptr {
    T* p;
    smart_ptr(T* x): p(x) {}
    ~smart_ptr() { delete p; };
};
```
or a lock_guard
```cpp
template <typename T>
struct lock_guard {
    T& m;
    lock_guard(T& x) : m(x) { m.lock(); };
    ~lock_guard() { m.unlock(); }
};
```

RAII is an idiomatic way for handling resources in C++, hence it is pervasive in the standard library: `string`, `vector`, `map`, `unordered_map`, `ifstream`, `ofstream`, `thread`, `lock_guard`, `unique_lock`, `unique_ptr` and `shared_ptr` all use RAII to achieve strong resource safety. RAII is usually a better alternative to explicitly acquiring and releasing resources. [2]

## 2. Smart Pointers
Smart pointers are essentially a slightly fancier version of `smart_ptr` described in previous section. It's an RAII wrapper around a resource which guarantees the resource is released when the object is destroyed.

The two most common smart pointers are `unique_ptr<T>` and `shared_ptr<T>`. The *unique* and *shared* part refer to the ownership of the underlying resource, not its use as is frequently assumed. The owner is the one responsible for releasing the resource when it's no longer needed. A `unique_ptr` can only have a single owner, so it requires the `unique_ptr` to be moved when transferring ownership. A `shared_ptr` can have multiple owners, each owner retains a copy of the `shared_ptr`. A function or a class can use a resource without owning it.

There are generally two ways to pass a resource, one indicating a change in ownership, and one merely granting access to use the resource. A change in ownership is accomplished by passing the smart pointer by value.
```cpp
void f(unique_ptr<T> x);
void f(shared_ptr<T> x);
```
For `unique_ptr`, the ownership will be transferred from the caller to `f`. In the case of `shared_ptr`, `f` will share ownership with the caller.

However, in most cases the function `f` doesn't require ownership, it simply needs to use the resource. In this case, `T` should be passed, either by reference or pointer.
```cpp
void f(T& x);
void f(T* x);
```
`const` should be added if the object is read only.

It's possible to pass `shared_ptr<T>` instead of `T&` or `T*` when `f` doesn't need ownership, but it's premature pessimization [3]
- it's confusing because the reader assumes that the function `f` will take shared ownership of the resource and continue to use it beyond `f`'s lifetime.
- `shared_ptr` does a synchronized increment/decrement of the reference count each time it is passed by value to a function. It's a pretty cheap but unnecessary performance penalty.
- if the caller of the function decides to manage the resource through a `unique_ptr`, or not use a pointer at all, the interface of `f` needs to change.

## 3. Dynamic Memory
C++ doesn't require the keyword `new` to create a new object. Using the keyword `new` puts the object on the heap. This may sound obvious, but it happens frequently. As seen in previous section, smart pointers offer a safe way to manage dynamic memory. However, dynamic memory is often unnecessary. In fact, the issues encountered in section 1 on RAII with managing the memory of the variable `x` in function `f`, would not occur had it not been for trying to unnecessarily use dynamic memory. There's no danger of leaking resources in the following function.
```cpp
void f(int i) {
    X x;
    if (i < x.value) return;
    x.do_something();
}
```
Allocating objects on the heap is often unnecessary, and the code is simpler without it. There are usually only two reasons to manage resources using smart pointers
- `shared_ptr<T>` when an object needs to have shared ownership, e.g. on two different threads with different lifetimes
- `unique_ptr<T>` (or a reference) when referring to polymorphic objects since the exact type or size of the object is unknown

A shared pointer is as good as a global variable. Mutable global variables are generally avoided because they introduce a global state, making it hard to locally reason about the code. The same goes for `shared_ptr`. The most common use of `shared_ptr` is bad design.

## 4. Wrappers
Wrappers around wrappers around wrappers are generally unnecessary. It's not uncommon see something like
```cpp
class MyClass {
  private:
    int x;
    string y;
    bool z;
  public:
    int get_x();
    string get_y();
    bool get_z();
    void set_x(int val);
    void  set_y(string val);
    void  set_z(bool val);
};
```
But it's simpler to use
```cpp
struct MyClass {
    int x;
    string y;
    bool z;
};
```
instead and access the data directly. Similarly, one way to implement a data structure that maps a string to any type could be something like
```cpp
class MyUnnecessaryAnyDataTypeAbstraction {
  private:
    any my_any_type;
  public:
    void put_int(int val);
    void put_string(string val);
    void put_bool(bool val);
    int get_int();
    string get_string();
    bool get_bool();
};

class MyUnecessaryMapDataWrapperAbstraction {
  private:
    map<string, MyUnnecessaryAnyDataTypeAbstraction> my_map;
  public:
    void put_int(string key, int val);
    void put_string(string key, string val);
    void put_bool(string key, bool val);
    int get_int(string key);
    string get_string(string key);
    bool get_bool(string key);
};
```
While this may earn good grades in an OO class, or good money if paid by LOC written, it's often clearer to just use `map<string, any>` directly. If a class is only inserting (extracting) data into (from) a data structure, just use the data structure. The user is usually more familiar with a `map<string, any>` than a `MyUnecessaryMapDataWrapperAbstraction`. In C++, the best abstraction of a map is a `std::map`.

Another way unnecessary wrappers manifest themselves is by a class wrapping a class wrapping a class, without really abstracting anything. A symptom of this is needing to dig through multiple layers of function calls in order to understand what's actually happening. For example:
```cpp
bool MyClass1::do_something(int i) {
    return my_class2.do_something_for_real(i);
}

bool MyClass2::do_something_for_real(int i) {
    return my_class3.do_something_for_real_for_real(i);
}

bool MyClass3::do_something_for_real_for_real(int i) {
    return true; // this function is always true
}
```
This might be a bit oversimplified, but there are a lot of functions that consist of a single line calling another function, often abstracting nothing.


## 5. Algorithms
There are numerous algorithms provided by C++'s standard library. It's worth knowing and using them. There's no point in reimplementing `find_if` or `copy`. A custom implementation is probably buggy and slower than what's in the standard library.

Sure, this works
```cpp
// Next, check if the panel has moved to the other side of another panel.
for (size_t i = 0; i < expanded_panels_.size(); ++i) {
    Panel* panel = expanded_panels_[i].get();
    if (center_x <= panel->cur_panel_center() || i == expanded_panels_.size() - 1) {
        if (panel != fixed_panel) {
            // If it has, then we reorder the panels.
            ref_ptr ref = expanded_panels_[fixed_index];
            expanded_panels_.erase(expanded_panels_.begin() + fixed_index);
            if (i < expanded_panels_.size()) {
                expanded_panels_.insert(expanded_panels_.begin() + i, ref);
            } else {
                expanded_panels_.push_back(ref);
            }
         }
         break;
    }
}
```
but this is probably better. [1]
```cpp
auto f = begin(expanded_panels_) + fixed_index;
auto p = lower_bound(begin(expanded_panels_), f, center_x,
    [](const ref_ptr& e, int x){ return e->cur_panel_center() < x; });
rotate(p, f, f + 1);
```
## References
1. [C++ Seasoning](https://youtu.be/W2tWOdzgXHA), Sean Parent
2. [A tour of C++](https://www.amazon.com/dp/0134997832), Bjarne Stroustrup
3. [Back to the Basics! Essentials of Modern C++ Style](https://youtu.be/xnqTKD8uD64), Herb Sutter
4. [GOTW #91](https://herbsutter.com/2013/06/05/gotw-91-solution-smart-pointer-parameters/), Herb Sutter
