# Mutex-Based Synchronization

When designing a high-performance system, one of the fundamental decisions to make is the memory model and synchronization strategy. Consider a simple multithreaded cache manager with a single interface `get(key) -> value`, where the key represents the file name and the value represents its content. The `get` operation involves checking if the entry is present in the hash table. If not, the system loads the content from disk, inserts it into the hash table, and then returns the content.

Even for this basic system, synchronization of access to the hash table is necessary. The pseudocode for such a system can be summarized as follows:

1. Check if the entry is present in the cache by
   - Locking a mutex
   - Checking if the entry is present in the hash table
   - Unlocking the mutex
2. If the entry was present, go to step 4. Otherwise, read the value from disk (possibly in a separate thread or thread pool).
3. Add the value to the cache
   - Lock the mutex
   - Add the value to the hash table
   - Unlock the mutex
4. Return the value

## Benchmarking Mutex Locking Performance

To simulate a similar pattern, we can perform the following operations:

1. Lock a mutex and perform CPU-intensive operations on a data structure, then unlock the mutex.
2. Perform IO/network operations that can be parallelized.
3. Lock the mutex again and perform additional operations on the data structure before unlocking the mutex.

In this blog, we will conduct a benchmark to observe how mutex locking affects performance. Subsequently, we plan to evaluate different libraries and alternatives.

### Simulating the Operation

Let's use the following sequence of operations in C++:


```cpp
using cachetype = std::unordered_map<std::string, std::string>;
const unsigned int max_iter = 1000;

// Simulate CPU operation by just incrementing a string value
// in a hash table
inline std::string get_key(int i) {return cache_key + std::to_string(i);}

inline void calc(cachetype& cache) {
    // Some complex calculation
    for (int i=0; i < num_calc; i++) {
        auto key = get_key(i);
        auto str = cache[key];
        cache[key] = str.length() == 0 ? "1" : std::to_string(std::stoi(str) + 1);
    }
}

// Simulate an  IO operation by just sleep
inline void fake_io (std::chrono::duratin sleep_time) {
    std::this_thread::sleep_for(sleep_time);
}

```


For the lock-free version of the benchmark:


```cpp
static void cache_calc(cachetype& cache,
    std::chrono::duratin sleep_time) {
    for (int i=0; i < max_iter; i++) {
        calc(cache);
        fake_io(sleep_time);
        calc(cache);
    }
}

```


Here, `sleep_time` simulates the I/O intensity of the application, and we will perform the benchmark with values of 8us and 64us.

The benchmark results for 1000 iterations:


```console
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
BM_cachecalc/8              85.0 ms        0.044 ms          100
BM_cachecalc/64              155 ms        0.044 ms          100
```


Adding mutex in a single-threaded program surprisingly shows no overhead, indicating minimal impact when there is no contention:



```cpp
static void cache_calc_mutex(cachetype& cache, std::mutex& m,
    std::chrono::microseconds sleep_time) {
    for (int i=0; i < max_iter; i++) {
        {
            std::scoped_lock lck(m);
            calc(cache);
        }
        fake_io(sleep_time);
        {
            std::scoped_lock lck(m);
            calc(cache);
        }
    }
}
```


Benchmark results with mutex:


```console
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
BM_cachecalc_mutex/8        84.8 ms        0.048 ms          100
BM_cachecalc_mutex/64        155 ms        0.042 ms          100

```


Running the same calculation in 8 threads yields the following benchmark results:



```cpp
constexpr unsigned int num_tasks = 8

static void cache_calc_async(cachetype& map,
    std::chrono::microseconds sleep_time) {
    std::queue<std::future<void>> futures;
    std::mutex m;

    for (int i=0; i < max_iter; i++) {
        futures.push(std::async([&]{
            {
                std::scoped_lock lck(m);
                calc(map);
            }
            fake_io(sleep_time);
            {
                std::scoped_lock lck(m);
                calc(map);
            }
        }));
        while (futures.size() >= num_tasks) {
            futures.front().wait();
            futures.pop();
        }
    }
    while(!futures.empty()) {
        futures.front().wait();
        futures.pop();
    }
}

```


Benchmark results for 8 threads:


```console
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
BM_cachecalc_async/8        17.5 ms         13.6 ms           48
BM_cachecalc_async/64       24.0 ms         15.4 ms           44
```


For the full code of the benchmark, you can visit [EvaluateIPC GitHub Repository](https://github.com/ahsank/EvaluateIPC/blob/master/tests/mutexbench.cc)
