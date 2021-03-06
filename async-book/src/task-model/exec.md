# A toy task executor

Let's build a task executor! Our goal is to make it possible to run any number
of tasks cooperatively on a *single* OS thread. To keep things simple, we'll opt
for the most naive data structures in doing so. Here's what we import:

```rust
use std::mem;
use std::collections::{HashMap, HashSet};
use std::sync::{Arc, Mutex, MutexGuard};
use std::thread::{self, Thread};
```

First up, we'll define a structure that holds the executor's state. The executor
owns the tasks themselves, and gives each task a `usize` ID so that the task can
be referred to externally. In particular, the executor keeps a `ready` set of
task IDs for tasks that should be woken up (because an event they were waiting
on has occurred):

```rust
// the internal executor state
struct ExecState {
    // The next available task ID.
    next_id: usize,

    // The complete list of tasks, keyed by ID.
    tasks: HashMap<usize, TaskEntry>,

    // The set of IDs for ready-to-run tasks.
    ready: HashSet<usize>,

    // The actual OS thread running the executor.
    thread: Thread,
}
```

The executor itself just wraps this state with an `Arc`ed `Mutex`, allowing it
to be *used* from other threads (even though all the tasks themselves will run
locally):

```rust
#[derive(Clone)]
pub struct ToyExec {
    state: Arc<Mutex<ExecState>>,
}
```

Now, the `tasks` field of `ExecState` provides `TaskEntry` instances, which box
up an actual task, together with a `Waker` for waking it back up:

```rust
struct TaskEntry {
    task: Box<ToyTask + Send>,
    wake: Waker,
}
```

Finally, we have a bit of boilerplate for creating and working with the executor:

```rust
impl ToyExec {
    pub fn new() -> Self {
        ToyExec {
            state: Arc::new(Mutex::new(ExecState {
                next_id: 0,
                tasks: HashMap::new(),
                ready: HashSet::new(),
                thread: thread::current(),
            })),
        }
    }

    // a convenience method for getting our hands on the executor state
    fn state_mut(&self) -> MutexGuard<ExecState> {
        self.state.lock().unwrap()
    }
}
```

With all of that scaffolding in place, we can start looking at the inner
workings of the executor. It's best to start with the core executor loop, which
for simplicity never exits; it just continually runs all spawned tasks to
completion:

```rust
impl ToyExec {
    pub fn run(&self) {
        loop {
            // Each time around, we grab the *entire* set of ready-to-run task IDs:
            let mut ready = mem::replace(&mut self.state_mut().ready, HashSet::new());

            // Now try to `complete` each initially-ready task:
            for id in ready.drain() {
                // We take *full ownership* of the task; if it completes, it will
                // be dropped.
                let entry = self.state_mut().tasks.remove(&id);
                if let Some(mut entry) = entry {
                    if let Async::Pending = entry.task.poll(&entry.wake) {
                        // The task hasn't completed, so put it back in the table.
                        self.state_mut().tasks.insert(id, entry);
                    }
                }
            }

            // We've processed all work we acquired on entry; block until more work
            // is available. If new work became available after our `ready` snapshot,
            // this will be a no-op.
            thread::park();
        }
    }
}
```

The main subtlety here is that, in each turn of the loop, we `poll` everything
that was ready *at the beginning*, and then "park" the thread.  The
[`park`]/[`unpark`] APIs in `std` make it very easy to handle blocking and
waking OS threads. In this case, what we want is for the executor's underlying
OS thread to block unless or until some additional tasks are ready. There's no
risk we'll fail to wake up: if another thread invokes `unpark` between our
initial read of the `ready` set and calling `park`, the call to `park` will
immediately return.

[`park`]: https://static.rust-lang.org/doc/master/std/thread/fn.park.html
[`unpark`]: https://static.rust-lang.org/doc/master/std/thread/struct.Thread.html#method.unpark

On the other side, here's how a task is woken up:

```rust
impl ExecState {
    fn wake_task(&mut self, id: usize) {
        self.ready.insert(id);

        // *after* inserting in the ready set, ensure the executor OS
        // thread is woken up if it's not already running.
        self.thread.unpark();
    }
}

struct ToyWake {
    // A link back to the executor that owns the task we want to wake up.
    exec: SingleThreadExec,

    // The ID for the task we want to wake up.
    id: usize,
}

impl Wake for ToyWake {
    fn wake(&self) {
        self.exec.state_mut().wake_task(self.id);
    }
}
```

The remaining pieces are then straightforward. The `spawn` method is
responsible for packaging up a task into a `TaskEntry` and installing it:

```rust
impl ToyExec {
    pub fn spawn<T>(&self, task: T)
        where T: Task + Send + 'static
    {
        let mut state = self.state_mut();

        let id = state.next_id;
        state.next_id += 1;

        let wake = ToyWake { id, exec: self.clone() };
        let entry = TaskEntry {
            wake: Waker::from(Arc::new(wake)),
            task: Box::new(task)
        };
        state.tasks.insert(id, entry);

        // A newly-added task is considered immediately ready to run,
        // which will cause a subsequent call to `park` to immediately
        // return.
        state.wake_task(id);
    }
}
```

And with that, we've built a task scheduler! But before we get too excited, it's
important to realize that as written, this implementation leaks tasks due to
`Arc` cycles. It's a good exercise to try to observe and fix this problem.

Still, this was intended as a toy executor after all. Let's move on to building
a source of events for tasks to wait on.
