@def hasdiagram = true

# Actor Framework in C++: Efficient Message Passing and Synchronization

In an actor-based framework, the responsibility for acting on a data structure requiring synchronization lies with a single thread. When other threads need to modify or read data, they send messages to the thread managing the data. These messages are delivered one at a time. It's assumed that operations on the data structure take a very short amount of time. If the managing thread needs to perform time-consuming I/O operations, they are done asynchronously to avoid blocking and waiting for I/O completion.

It can be shown as following diagram:

Diagram:

~~~
<div class="mermaid">
 graph LR
      A[Thread1] -->Queue
      E[Thread2] -->Queue
      Queue --> B[fa:fa-ban Actor Thread]
      B <-->D(fa:fa-spinner Data structure);
</div>
~~~


## Advantages and Challenges

The advantage of such a system is that the data structure can be accessed by a single thread straightforwardly without the need for locks. However, the challenging factor lies in implementing an efficient message-passing system.

To benchmark this system, we'll use a computational model discussed in the blog post [Benchmarking mutex-based synchronization](/posts/MutexBasedSynchronization/). This model interleaves CPU and I/O operations, where the CPU operation `calc(cache)` works on a hash table that requires synchronization, and the simulated I/O operation `fake_io(sleep_time)` just sleeps for a specified duration. We assume that the I/O operation can be parallelized without imposing an additional burden on the CPU. The benchmark will be run for sleep times of 8 and 64 microseconds.


```cpp
// Code snippet for benchmark common functions
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

First let's try to build an actor framework using standard C++ library and later
we will use the boost library to implment a another version.


## Simple Multithreaded Queue

A simple multithreaded queue, with `add` and get operations, forms the basis for our actor framework. The add operation waits until the queue has capacity and then adds to the end, while the get operation waits until the queue has elements and removes an entry from the beginning.



```cpp
template <class T, int qsize=100>
class cond_queue {
public:
    std::queue<T> buff;
    volatile bool closed = false;
    // See https://en.cppreference.com/w/cpp/thread/condition_variable
    std::mutex m_mutex;
    std::condition_variable m_cv;

    cond_queue() {
    }
    void add(T val) {
        {
            std::unique_lock<std::mutex> the_lock(m_mutex);
            while (buff.size() >= qsize) {
                m_cv.wait(the_lock, [this] {
                    return buff.size() < qsize;
                });
            }
            buff.push(std::move(val));
        }
        m_cv.notify_one();
    }

    void close() {
        {
            std::unique_lock<std::mutex> the_lock(m_mutex);
            closed = true;
        }
        m_cv.notify_all();
    }
    
    std::pair<T, bool> get() {
        T val;
        bool is_closed = false;
        {
            std::unique_lock<std::mutex> the_lock(m_mutex);
            while ( buff.empty() && !closed) {
                m_cv.wait(the_lock, [this] {
                    return !buff.empty() || closed;
                });
            }

            if (!buff.empty()) {
                val = std::move(buff.front());
                buff.pop();
            } else {
                is_closed = closed;  
            }
        }
        m_cv.notify_one();
        return std::make_pair(std::move(val), is_closed);
    }
};

```

Using this queue, we implement an actor-based class for our hashtable.

```
class CacheActor {
public:
    cachetype cache;
    using tasktype = std::packaged_task<void(cachetype&)>;
    cond_queue<tasktype> cqueue;
    std::thread actor_thread;

    void start() {
        actor_thread = std::thread([&] {
            while (true) {
                auto [task, closed] = cqueue.get();
                if (closed) {
                    break;
                }
                task(cache);
            }
        });
    }

    void stop() {
        cqueue.close();
        actor_thread.join();
    }

    std::future<void> do_calc() {
        tasktype calctask(calc);
        auto fut = calctask.get_future();
        cqueue.add(std::move(calctask));
        return fut;
    }
};

```


Note that operations on the hash table are serialized by the queue and executed by a single thread. We run the benchmark with 1000 parallel tasks, and the calculation operations are done through the actor framework, serializing the operation.

Following is the function to benchmark:

```
void cache_calc_queue(CacheActor& actor,
    std::chrono::microseconds sleep_time) {
    actor.start();
    std::vector<std::future<void>> futures;
    for (int i=0; i < max_iter; i++) {
        futures.push_back(std::async(std::launch::async,
                [&actor, sleep_time] {
                    actor.do_calc().wait();
                    fake_io(sleep_time);
                    actor.do_calc().wait();
                }));
    }
    for(auto& f : futures) {
        f.wait();
    }
    actor.stop();
}

```


### Benchmark results


```console
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
BM_cachecalc_queue/8             23.6 ms         21.9 ms           30
BM_cachecalc_queue/64            23.3 ms         21.7 ms           32
```

Comparing to the previous benchmark, we see significant performance improvements.


## Boost Implementation

Now, let's explore a boost-based implementation of the same benchmark using a [strand](https://live.boost.org/doc/libs/1_84_0/doc/html/boost_asio/overview/core/strands.html) to serialize access.


```cpp
static void BM_cachecalc_threadpool(benchmark::State& state) {
    boost::asio::thread_pool pool(num_tasks);
    boost::asio::thread_pool calcpool(1);
    auto strand_ = make_strand(calcpool.get_executor());
    std::chrono::microseconds sleep_time = std::chrono::microseconds(state.range(0));
    for (auto _ : state) {
        cachetype cache;
        std::vector<boost::future<void> > futures;
        futures.reserve(max_iter);
        auto fn = [&] {
            auto f = boost::asio::post(strand_, std::packaged_task<void()>([&] {
                calc(cache);
            }));
            f.wait();
            fake_io(sleep_time);
            auto f1 = boost::asio::post(strand_, std::packaged_task<void()>([&] {
                calc(cache);
            }));
            f1.wait();
        };
        for (int i=0; i < max_iter; i++) {
            boost::packaged_task<void()> task(fn);
            futures.push_back(task.get_future());
            boost::asio::post(pool, std::move(task));
        }
        boost::wait_for_all(futures.begin(), futures.end());
        checkWork(state, cache);
    }
}

```


### Benchmark Resuls

Following is the benchmark result of boost based implemenatation

```console
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
BM_cachecalc_threadpool/8        17.1 ms         2.68 ms          274
BM_cachecalc_threadpool/64       27.2 ms         3.58 ms          100
```

This boost-based implementation shows some performance gain compared to our basic implementation.

The complete code for the benchmarks can be found [here](https://github.com/ahsank/EvaluateIPC/blob/master/tests/boostbench.cc).


{{ addcomments }}
