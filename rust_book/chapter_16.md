# Notes on Fearless Concurrency: Chapter 16 of The Rust Book

Concurrent programming: diferent parts of the program execute independently
Parallel programming: different parts of the program execute at the same time

Rust catches errors before run time, avoiding race conditions during compile time

State needs to be shared between threads

Coverage:
* How to create threads to run multiple pieces of code at the same time
* Message-passing concurrency, where channels send messages between threads
* Shared-state concurrency, where multiple threads have access to some piece of data
* The Sync and Send traints, which extend Rust's concurrency guarantees to user-defined types as well as types provided by the standard library

## Creating threads

Threads are the features of a process that run simultaneously and independently

There is no guarantee about the order in which parts of your code on different threads will run, leading to:
* Race conditions: threads access data or resources in an inconsistent order
* Deadlocks: two threads waiting for each other, preventing both threads from continuing
* Bugs

The Rust standard library uses a 1:1 model of thread implementation in which the program uses one operating systm thread per one language thread.

Creating a thread:
```
let handle = thread::spawn(|| {

})
```

When the main thread of a Rust program completes, all spawned threads are shut down, whether or not they have finished running

Calling join on the handle blocks the thread currently running until the thread represented by the handle terminates:
```
handle.join().unwrap()
```

Small details, such as where join is called, can affect whether or not your threads run at the same time

The move keyword is required to give the thread ownership of a variable declared in the main thread:
```
let handle = thread::spawn(move || {

})
```

## Using Message Passing to Transfer Data Between Threads

A channel is a general programming concept by which data is sent from one thread to another

A channel has two halves: a transmitter and a receiver

A channel is said to be closed if either the transmitter or receiver half is dropped

One thread generating values and sending them through a channel and another thread receiving the values:

You could also use channels for any threads that need to communicate with each other such as a chat system or map reduce operation with a coordinator master thread

```
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });
    let received = rx.recv().unwrap();
    println!("Got: {received}");
}
```

mpsc stands for multiple producer, single consumer

The mpsc::channel function returns a tuple, the first element of which is the sending end--the transmitter--and the second element of which is the receiving end--the receiver. tx and rx is conventional abbreviations for transmitter and receiver.

The spawned thread needs to own the tx to be able to send messages through the channel
The transmitter has a send method that takes the value we want to send, and returns a Result<T, E> type, so it the receiver has already been dropped and there's nowhere to send a value, the send operation will return an error.

unwrap is not best practice here but used for convenience

recv blocks the main thread's execution and wait until a value is sent down the channel

try_recv is useful if the receiving thread has other workto do while waiting for messages: wecould write a loop that calls try_recv every so often, handles a message if one is available, and otherwise does other work for a little while until checking in again.

**Channels and Ownership Transfer**
After sending a value down a channel through tx.send, the value has been moved and ownership transferred to the receiver; this stops us from accidentally using the value again after sending it


```
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    thread::spawn(move || {
        let vals = vec![
            String::from("hi");
            String::from("from");
            String::from("the");
            String::from("thread");
        ];
        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });
    for received in rx {
        println!("Got: {received}");
    }
}
```

We treated rx as an iterator
In the main thread, we're not calling the recv function explicitly anymore; instad we're treating rx as an iterator. For each value recived, we're printing it. When the channel is closed, iteration will end.

**Creating Multiple Producers by Cloning the Transmitter**
`let tx1 = tx.clone();`
The clone can be used by the first thread and the actual tx can be used by the second thread to send values down the same channel


## Shared-State Concurrency
Another method is for multiple thread so access the same shared data

Go language documentation: "Do not communiate by sharing memory"
Channels in any programming language are similar to single ownership; once you transfer a value down a channel, you should no longer use that avlue
Shared-memory concurrency is like multiple ownership; multiple threads can access the same memory location at the same time

Multiple ownership can add complexity because different owners need managing; mutexes are a common concurrency synchronization primitive for shared memory

**Using Mutexes to Allow Access to Data from One Thread at a Time**

mutex for mutual exclusion, allow only one thread to access some data at any given time

The lock is a data structure that is part of a mutex that keeps track of who currently has exclusive access to the data

Two rules for using mutexes:
1. You must attempt to acquire the lock before using the data
2. When you're done with the data that the mutex guards, you must unlock the data so other threads can acquire the lock

```
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(S);
    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }
    println!("m = {:?}", m);
}
```

A mutex is creatd using the function called new and to access the data inside the mutex, we use the lock method to acquire the lock

After dropping the lock, we can print the mutex value and see that we were able to change the inner i32 to 6.

Mutex<T> is a smart pointer. The call to lock returns a smart pointer called MutexGuard, wrapped in a lockResult that we handled with the call to unwrap. The MutexGuard, wrapped in a LockResult that we handled with the call to unwrap. The MutexGuard smart pointer implements `Deref` to point at our inner data; the smart pointer also has a `Drop` implementation that releases the lock automatically when a MutexGuard goes out of scope, which happens at the end of the inner scope. As a result, we don't risk forgetting to release the lock and blocking the mutex from being used by other threads because the lock release happens automatically.

The call to the lock would fail if another thread holding the lock panicked. In that case, no one would ever be able to get the lock, so we've chosen to unwrap and have this thread panic if we're in that situation.

### Sharing a Mutex<T> Between Multiple Threads

Spinning up 10 threads and have them each increment a counter value by 1.

The following will not work because we can't move the ownership of lock counter into multiple threads
```
use std::sync::Mutex;
use std::thread;

fn main () {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..10 {
        let handle = thread::spawn( move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }
    // num goes out of scope here, so the lock is released
    for handle in handles {
        handle.join().unwrap();
    }
    println!("Result: {}", *counter.lock().unwrap());
}
```

We need to use the multiple-ownership method, using the smart pointer Rc<T> to create a reference counted value.

We could try to wrap the Mutex<T> in Rc<T> and clone the Rc<T> before moving ownership to the thread, but Rc<T> is not thread safe. `Rc<Mutex<i32>>` does not implement the `Send` trait, which is one of the traits that ensures the types we use with threads are meant for use in concurrent situations.

### Atomic Reference Counting with Arc<T>

```
use std::sync::{Arc, Mutex};
use std::thread;

fn main () {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter); // This is a shadow of the previous counter (part of loop), and clones counter from the outer scope
        let handle = thread::spawn( move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }
    // num goes out of scope here, so the lock is released
    for handle in handles {
        handle.join().unwrap();
    }
    println!("Result: {}", *counter.lock().unwrap());
}
```

You could also do more complicated operations than just incrementing a counter, including divide a calculation into independent parts, split those parts across threads, and then use a Mutex<T> to have each thread update the final result with its part.

Note that if you are doing simple numerical operations, there are types
simpler than Mutex<T> types provided by the std::sync::atomic module of
the standard library. These types provide safe, concurrent, atomic access
to primitive types. We chose to use Mutex<T> with a primitive type for this
example so we could concentrate on how Mutex<T> works.

## Extensible Concurrency with the Send and Sync Traits

Two concurrency concepts are embedded in the Rust language: the `std::marker` traits `Send` and `Sync`

### Allowing Transference of Ownership Between Threads with Send
The `Send` marker trait indicates that ownership of values of the type implementing `Send` can be transferred between threads. Almost every Rust type is `Send`, but there are some exceptions, including `Rc<T>`: this cannot be `Send` because if you cloned an Rc<T> value and tried to transfer ownership of the clone to another thread, both threads might update the reference count at the same time. For this reason, `Rc<T>` is implemented for use in single-threaded situations where you don't want to pay the thread-safe performance penalty.

Any type composed entirely of `Send` types is automatically marked as `Send` as well. Almost all pimitive types are `Send`, aside from raw pointers.

### Allowing Access from Multiple Threads with Sync
The `Sync` marker trait indicates that it is safe for the type implementing Sync to be referenced from multiple threads. Any type `T` is `Sync` if &T (an immutable reference T) is `Send`, meaning the reference can be sent safely to another thread. similar to `Send`, primitive types are `Sync`, and types composed entirely of types that are `Sync` are also `Sync`.

Smart pointers such as Rc<T> and RefCell<T> and the family of related Cell<T> types are not Sync.

The smart pointer `Mutex<T>` is `Sync` and can be used to share access with multiple threads, as shown above.

### Implementing Send and Sync Manually is Unsafe
Because types that are made up of `Send` and `Sync` traits are automatically also `Send` and `Sync`, we don't have to implement those traits manually. As marker traits, they don't even have any methods to implement. They're just useful for enforcing invariants related to concurrency.

Manually implementing these traits involves implementing unsafe Rust code. Building new concurrent types not made up of `Send` and `Sync` parts requires careful thought to uphold the safety guarantees.
