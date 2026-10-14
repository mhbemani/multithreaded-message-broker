# multithreaded-message-broker

A priority-based publish/subscribe message broker written in C11 with POSIX threads. Producers publish messages to topics, a dispatcher per topic routes them to consumers, and the whole thing runs as a self-contained simulation that prints what every thread is doing. Built as an operating-systems course project on synchronization.

## How it works

```
producers --> [ topic queue (3 priority levels) ] --> dispatcher --> [ consumer queue ] --> consumers
                                                         ^
                                              GC thread (expired messages)      dashboard thread (stats)
```

- **Producers** create messages with a priority (high / medium / low, chosen by weights), an estimated processing time, an optional deadline and a TTL.
- **Topics** hold a priority queue guarded by a mutex and two condition variables. In bounded mode a full queue blocks producers.
- **Dispatchers** (one per topic) pick the consumer that is expected to finish the message best, using each consumer's queued load per priority and the message's deadline. A message that cannot make its deadline is rejected. In fanout mode every subscriber gets a copy instead.
- **Consumers** process messages (simulated by sleeping for the estimated time) and keep per-priority statistics.
- **GC thread** removes messages whose TTL has expired (`--retention`).
- **Dashboard thread** prints queue sizes, consumer load and latency totals once in a while.
- Shutdown sets an atomic stop flag and wakes every waiter under its own mutex, so no thread is left sleeping.

## Build and run

```
gcc -std=c11 -O2 -pthread -o broker multithreaded_message_broker.c
./broker
./broker --seconds=10 --producers=3 --consumers=3 --topics=2
```

| Flag | Meaning | Default |
|---|---|---|
| `--seconds=N` | run time | 5 |
| `--producers=N` | producer threads | 5 |
| `--consumers=N` | consumer threads | 5 |
| `--topics=N` | number of topics | 3 |
| `--bounded` | bounded topic and consumer queues | off |
| `--capacity=N` | queue capacity when bounded | 0 |
| `--retention` | drop expired messages (TTL) | off |
| `--ttl=N` | message time-to-live in ms | 500 |
| `--gc-interval=N` | how often the GC thread runs, in ms | 800 |
| `--fanout` | publish/subscribe: every consumer of a topic gets every message | off |
| `--prio-weights=H,M,L` | weights for generating high/medium/low messages | 1,3,6 |

`--producers` and `--consumers` must each be between `topics` and `2 * topics`.

## Sample result

`./broker --seconds=10 --producers=3 --consumers=3 --topics=2` ends with a summary like:

```
Done. Rejected=62
Processing totals: H=31 M=78 L=18
End-to-end latency totals: H=9426ms M=49830ms L=58847ms
```

## Testing

Built with `-fsanitize=thread` and run with the default, bounded and fanout/retention configurations: ThreadSanitizer reports no data races. There are no automated tests; the simulation output is the check.

## Known limitations

- Messages still sitting in a queue at shutdown are dropped without being counted.
- Consumer queues do not expire messages; only topic queues are cleaned by the GC thread.
- `rand()` is shared by all producer threads.
