# Mutex-Based Synchronization: Understanding Performance Impacts

When designing a high-performance systems, an early decision is to select the appropriate memory model and synchronization strategy. Consider a simple multithreaded cache manager with a single interface: `get(key) -> value`, where the key represents the file name and the value represents its content. Here the `get` operation checks if the entry is present in the hash table. If not, then loads the content from disk, inserts it into the hash table, and then returns the content.

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

In this blog, I will do benchmarks to find how mutex locking affects performance. Subsequently, I plan to evaluate different libraries and alternatives of locking.


### Simulating IO Operations for Benchmarking

The benchmark time  heavily depends on the IO library and the intricacies of the computer system. In this blog, my primary goal is not to compare different IO libraries; rather, to measure the performance impact of the locking mechanism. To simplify simulating IO operations I initially thought of just using the sleep function. However then found that the minimum time duration the API could reliably sleep (without oversleeping) was approximately 1 millisecond, which seems impractical for mimicking realistic loads.

To determine a more realistic IO time, I did an experiment writing an 8KB memory to disk, which completed in less than 10 microseconds. Reading from disk was even faster. Therefore an IO time ranging from 10 to 100 microseconds seemed reasonable for our benchmarking purposes.

To exemplify, if I benchmark the following function which just sleeps for 8 and 64 microseconds:

```cpp
struct sleep_io {
    static inline void fake_io(std::chrono::microseconds sleep_time) {
        std::this_thread::sleep_for(sleep_time);
    }
};

void BM_sleep(benchmark::State& state) {
    auto duration = std::chrono::microseconds(state.range(0));
    for (auto _ : state) {
        sleep_io::fake_io(duration);
    }
}

BENCHMARK(BM_sleep)->Unit(benchmark::kMicrosecond)->Range(8, 64);
```

The benchmark results indicate that the actual time slept was considerably larger than the requested durations.

Benchmark        |             Time     |       CPU | Iterations
-----------------|----------------------|-----------|-----------
`BM_sleep/8`       |          83.6 us     |   4.25 us |     100000
`BM_sleep/64`      |           154 us     |   4.41 us |     100000


This prompted me to develop  a custom fake_io operation operation. It just reads from `/dev/zero` so will only work in Linux and Mac OSX.

```
// More accurate sleep
struct devzero_io {
    static inline void fake_io(std::chrono::microseconds sleep_time) {
        if (sleep_time <= 0us) return;
        using clock =  std::chrono::steady_clock;
        auto end_time = clock::now() + sleep_time;
        std::ifstream zfs("/dev/zero");
        const int readsize = 1024;
        char v[readsize];
        while ( clock::now() < end_time) {
            zfs.read(v, readsize);
        }
        zfs.close();
    }
};

using iotype = devzero_io;

```

Benchmarking this simulated IO operation, I got results close to the specified parameters, validating its effectiveness.


```
void BM_sleep_devzero(benchmark::State& state) {
    auto duration = std::chrono::microseconds(state.range(0));
    for (auto _ : state) {
        iotype::fake_io(duration);
    }
}

BENCHMARK(BM_sleep_devzero)->Unit(benchmark::kMicrosecond)->Range(8, 64);

```


Benchmark            |         Time     |       CPU | Iterations
---------------------|------------------|-----------|-----------
`BM_sleep_devzero/8`   |      8.91 us     |   8.87 us |      75051
`BM_sleep_devzero/64`  |      64.8 us     |   64.8 us |      10818


After determining the strategy for IO operation simulation, I used a CPU-intensive operation that increments an integer value and saves the value in a hash table as a string. This operation is representative of scenarios where data structure synchronization is necessary.


```cpp
using cachetype = std::unordered_map<std::string, std::string>;
const unsigned int max_iter = 1000;
const std::string cache_key = "hello";
const unsigned int num_calc = 10;
constexpr int start_val = 10000;

// Simulate CPU operation by just incrementing a string value
// in a hash table
inline std::string get_key(int i) {return cache_key + std::to_string(i);}

inline void calc(cachetype& cache) {
    // Some complex calculations
    for (int i=0; i < num_calc; i++) {
        auto key = get_key(i);
        auto str = cache[key];
        // converts to int, increments and converts back to string
        cache[key] = str.length() == 0 ?
            std::to_string(start_val+1) : std::to_string(std::stoi(str) + 1);
    }
}

```


Following is the code for the single threaded lock-free version of the benchmark:

```cpp
static void cache_calc(cachetype& cache,
    std::chrono::duratin sleep_time) {
    for (int i=0; i < max_iter; i++) {
        calc(cache);
        iotype::fake_io(sleep_time);
        calc(cache);
    }
}

```


Here, `sleep_time` simulates the I/O intensity of the application, and we will perform the benchmark with values of 8us and 64us.

The benchmark results for 1000 iterations:


Benchmark          |           Time    |        CPU | Iterations
-------------------|-------------------|------------|-----------
`BM_cachecalc/8`     |        10.1 ms    |    10.0 ms |         68
`BM_cachecalc/64`    |        65.8 ms    |    65.8 ms |         11



Adding mutex in a single-threaded program surprisingly shows no overhead, indicating minimal impact when there is no contention:


```cpp
void cache_calc_mutex(cachetype& cache, std::chrono::microseconds sleep_time) {
    std::mutex m;
    for (int i=0; i < max_iter; i++) {
        {
            std::scoped_lock lck(m);
            calc(cache);
        }
        iotype::fake_io(sleep_time);
        {
            std::scoped_lock lck(m);
            calc(cache);
        }
    }
}

```


Benchmark results with mutex:


Benchmark               |      Time    |        CPU | Iterations
------------------------|--------------|------------|-----------
`BM_cachecalc_mutex/8`    |   10.3 ms    |    10.2 ms |         68
`BM_cachecalc_mutex/64`   |   66.7 ms    |    66.7 ms |         11


The code and benchmark results for distributing work among 8 threads are as follows:


```cpp
constexpr unsigned int num_tasks = 8

static void cache_calc_async(cachetype& map, std::chrono::microseconds sleep_time) {
    std::queue<std::future<void>> futures;
    std::mutex m;

    for (int i=0; i < max_iter; i++) {
        futures.push(std::async([&]{
            {
                std::scoped_lock lck(m);
                calc(map);
            }
            iotype::fake_io(sleep_time);
            {
                std::scoped_lock lck(m);
                calc(map);
            }
        }));
        // Make sure no more parallel tasks than num_tasks
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

Benchmark               |      Time    |        CPU | Iterations
------------------------|--------------|------------|-----------
`BM_cachecalc_async/8`    |   10.0 ms    |    9.66 ms |         66
`BM_cachecalc_async/64`   |   23.6 ms    |    12.0 ms |         61



For the full code of the benchmark, you can visit [EvaluateIPC GitHub Repository](https://github.com/ahsank/EvaluateIPC/blob/master/tests/mutexbench.cc)

{{ addcomments }}
