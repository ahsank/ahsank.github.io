## Benchmarking Seastar: Unleashing Asynchronous Performance

This blog post explores the benefits of Seastar, an asynchronous framework, using our benchmark. Compared to traditional synchronous approaches, Seastar promises significant performance gains.

**Seastar: Unleashing Parallelism**

Seastar's core lies in its asynchronous system design. Each thread owns and guards its data segment, communicating with others through messages. This eliminates the need for locks and unlocks, boosting speed and efficiency.

**Futures and Continuations: Building Asynchronous Workflows**

Seastar revolves around two key concepts:

- **Futures:** These represent results waiting to happen, returned by asynchronous functions. You can wait for futures with `get()` or chain actions with `then()` for a linear, callback-free workflow.
- **Continuations:** These allow seamless chaining of actions after a future resolves, eliminating complex nested callbacks and state management.

**Seastar Benchmark: Putting Theory into Practice**

To showcase Seastar's power, we'll replicate a previous benchmark conducted with mutex-based synchronization. While Seastar offers a sophisticated Sharded Service for data management, we'll utilize the lower-level `seastar::smp::submit_to(...)` to run our `calc(cache)` function on a dedicated core, ensuring lock-free execution.

**Benchmark Results: Quantifying the Asynchronous Advantage**

| Benchmark          | Time (ms) | CPU (ms) | Iterations |
|--------------------|-----------|----------|------------|
| BM_SeastarHash/8   | 1.22      | 1.22      | 573        |
| BM_SeastarHash/64  | 1.25      | 1.25      | 544        |

The results are striking: Seastar outperforms the mutex-based benchmark by an order of magnitude!

Following is the snippet of the benchmark code. The build and run instructions can be found in [github repo]( https://github.com/ahsank/EvaluateIPC/tree/master/tests/seastar)


```cpp
int nshard = std::min(num_tasks, seastar::smp::count);
std::chrono::microseconds sleep_time = std::chrono::microseconds(state.range(0));
for (auto _ : state) {
    cachetype cache;
    seastar::parallel_for_each(boost::irange(0u, max_iter),
        [nshard, &cache, sleep_time] (unsigned f) {
             return seastar::smp::submit_to(0, [&cache] {
                 calc(cache);
                 return seastar::make_ready_future<>();
             }).then([sleep_time, f, nshard] {
                 return seastar::smp::submit_to(f % nshard, [sleep_time] {
                     return seastar::sleep(sleep_time);
                 });
             }).then([&cache] {
                 return seastar::smp::submit_to(0, [&cache] {
                     calc(cache);
                     return seastar::make_ready_future<>();
                 });
             });
    }).get();
    checkWork(state, cache);
}
```

**Conclusion: Embracing Asynchronous Power**

Seastar's innovative architecture and asynchronous approach make it a compelling choice for high-performance server applications. By leveraging its capabilities, developers can unlock exceptional speed and efficiency, shaping the future of lightning-fast applications.

**Further Exploration:**

- Explore Seastar's Sharded Service class for efficient multi-core data management.
- Consider Seastar's high-performance networking options for additional optimization.
- Dive deeper into Seastar's shared-nothing design for a comprehensive understanding of its architecture.

{{ addcomments }}
