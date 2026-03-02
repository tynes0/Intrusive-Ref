# Intrusive Reference Counting System

This library provides a high-performance, thread-safe intrusive reference counting system for C++. It is designed to manage object lifetimes automatically using strong and weak references, similar to `std::shared_ptr` and `std::weak_ptr`, but with the reference counters embedded directly within the object itself (intrusive). This avoids the secondary allocation overhead typical of standard library smart pointers.

## Core Components

The system consists of three main components:
1. `RefCountedObject`: The base class that holds the atomic counters.
2. `Ref<T>`: A strong reference smart pointer.
3. `WeakRef<T>`: A weak reference smart pointer that does not prevent object destruction.

---

## 1. `RefCountedObject`

To use the reference counting system, your classes must publicly inherit from `RefCountedObject`. This class maintains two counters:
* **Strong Count (`m_Strong`)**: Tracks the number of active `Ref<T>` instances owning the object.
* **Weak Count (`m_Weak`)**: Tracks the number of `WeakRef<T>` instances observing the object. It is initialized to `1` to represent the collective ownership of the strong references over the object's memory.

### Key Methods:
* `IncStrong() / DecStrong()`: Atomically modifies the strong reference count.
* `IncWeak() / DecWeak()`: Atomically modifies the weak reference count.
* `StrongCount() / WeakCount()`: Returns the current reference counts.
* `TryIncStrong()`: Atomically attempts to increment the strong count only if it is greater than zero. This is crucial for safely converting a `WeakRef` to a `Ref`.
* `virtual void OnZeroStrong()`: A virtual callback invoked exactly once when the strong reference count reaches zero, right before the object is considered logically dead.

---

## 2. `Ref<T>`

`Ref<T>` is a smart pointer that holds a strong reference to an object deriving from `RefCountedObject`. As long as at least one `Ref<T>` points to the object, the object will not be destroyed.

### Features:
* **Construction & Assignment**: Supports copy and move semantics. Moving transfers ownership without touching atomic counters.
* **Polymorphism**: Supports implicit conversion from `Ref<Derived>` to `Ref<Base>`.
* **Casting**:
    * `As<U>()`: Performs a `dynamic_cast` safely.
    * `AsFast<U>()`: Performs a `static_cast` (requires type safety assurance from the caller).
* **Memory Management**: When the last `Ref<T>` is destroyed, it triggers `OnZeroStrong()` and decrements the weak count. If the weak count also hits zero, the memory is deallocated.

### Example:
```cpp
class MyEntity : public RefCountedObject {
public:
    MyEntity() { /* ... */ }
    ~MyEntity() override { /* ... */ }
};

// Creating a strong reference
Ref<MyEntity> entity = Ref<MyEntity>::Create();

// Passing by value increments strong count
void ProcessEntity(Ref<MyEntity> e) {
    // e.StrongCount() is at least 2 here
}
```

## 3. `WeakRef<T>`

`WeakRef<T>` is a smart pointer that holds a non-owning ("weak") reference to an object managed by `Ref<T>`. It allows you to observe the object without preventing its logical destruction when all strong references are gone. This is particularly useful for breaking circular references (e.g., in parent-child node relationships or event listener systems) which would otherwise cause memory leaks.

### Features:
* **Creation**: A `WeakRef` is typically created from an existing `Ref<T>` or another `WeakRef<T>`.
* **Safe Access (`Lock`)**: You cannot directly dereference or use the `->` operator on a `WeakRef`. To access the underlying object, you must call the `Lock()` method.
    * If the object is still alive (i.e., its strong count is greater than 0), `Lock()` safely upgrades the weak reference to a strong `Ref<T>`, ensuring the object won't be destroyed while you are using it.
    * If the object has already been logically destroyed, `Lock()` safely returns a null `Ref<T>`.
* **Concurrency Safe**: Because it utilizes the `TryIncStrong()` atomic operation via a Compare-And-Swap (CAS) loop, locking is inherently thread-safe. It prevents race conditions where an object might begin its destruction sequence exactly as another thread attempts to access it.

### Example:
```cpp
Ref<MyEntity> strongEntity = Ref<MyEntity>::Create();
WeakRef<MyEntity> weakEntity = strongEntity;

// The strong reference is dropped, triggering logical destruction
strongEntity.Reset(); 

// Attempting to access the entity later
if (Ref<MyEntity> locked = weakEntity.Lock()) {
    // Object is still alive, safe to use 'locked'
} else {
    // The object's strong count reached zero, safely handle the null case
    // (This path will be executed in this specific example)
}
```

## Lifecycle & Memory Management

A critical distinction in this intrusive reference counting system—compared to standard library smart pointers like `std::shared_ptr`—is that the control block (the counters) and the object data are stored in the exact same memory allocation. Therefore, the physical memory of the object cannot be freed, and its C++ destructor `~T()` cannot be executed, until **all** weak references are completely gone.

To handle this, the lifecycle is split into two distinct phases: **Logical Destruction** and **Physical Destruction**.

1. **Initialization**:
    * When `RefCountedObject` is constructed, its internal counters begin at `m_Strong = 0` and `m_Weak = 1`.
    * The initial `m_Weak = 1` acts as a sentinel value. It represents the collective "hold" that all strong references have over the physical memory block.
    * When wrapped in the first `Ref<T>`, `m_Strong` is incremented to `1`.

2. **Logical Destruction (Strong Count Reaches 0)**:
    * When the last `Ref<T>` is destroyed or reset, `m_Strong` drops to `0`. The object is now considered "logically dead."
    * The system immediately fires the virtual method `OnZeroStrong()`. **Crucial Note:** Because the actual C++ destructor `~T()` is deferred, you must use `OnZeroStrong()` to release any heavy resources, close file handles, or break cyclic dependencies.
    * After `OnZeroStrong()` returns, the system decrements `m_Weak` by `1` (releasing the strong references' collective hold on the memory).

3. **Physical Destruction (Weak Count Reaches 0)**:
    * If there are no active `WeakRef` instances, `m_Weak` drops to `0` immediately after the logical destruction phase.
    * If `WeakRef` instances do exist, `m_Weak` remains `> 0`, keeping the entire object in memory so the weak references can safely check the counters.
    * As the remaining `WeakRef`s are eventually destroyed, they decrement `m_Weak`. When the absolute last `WeakRef` is destroyed and `m_Weak` finally reaches `0`, the physical destruction occurs: the C++ destructor `~T()` is executed, and `::operator delete` frees the raw memory footprint.

---

## Thread Safety Model

The library provides high-performance thread safety by utilizing `std::atomic` for all reference counter modifications. It avoids heavy mutexes by using precise memory ordering constraints:

* **Relaxed Increments**:
  Incrementing counters (`IncStrong`, `IncWeak`) uses `std::memory_order_relaxed`. Since incrementing merely states "I hold a reference," it does not require synchronizing the object's internal state across different CPU cores.
* **Synchronized Decrements (Acquire/Release)**:
  Decrementing counters uses `std::memory_order_acq_rel` (and `std::memory_order_acquire` for strong count reads). This acts as a critical memory barrier. It guarantees that any modifications made to the object's data by thread A are fully visible to thread B before thread B executes `OnZeroStrong()` or `~T()`.
* **Safe Weak-to-Strong Upgrades**:
  The `TryIncStrong` method (used internally by `WeakRef::Lock`) employs a Compare-And-Swap (CAS) loop with `compare_exchange_weak`. It strictly ensures that a weak reference cannot accidentally resurrect an object that is simultaneously in the process of dropping its strong count to zero on another thread.
