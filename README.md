*Note, Linux AIO is now subsumed by the io_uring API ([tutorial](https://blogs.oracle.com/linux/an-introduction-to-the-io_uring-asynchronous-io-framework), [LWN coverage](https://lwn.net/Articles/810414/)). The below explanation is mostly useful for old kernels.*

## Introduction
The Asynchronous Input/Output (AIO) interface allows many I/O requests to be submitted in parallel without the overhead of a thread per request. The purpose of this document is to explain how to use the Linux AIO interface, namely the function family `io_setup`, `io_submit`, `io_getevents`, `io_destroy`. Currently, the AIO interface is best for `O_DIRECT` access to a raw block device like a disk, flash drive or storage array.

## What is AIO?
Input and output functions involve a device, like a disk or flash drive, which works much slower than the CPU. Consequently, the CPU can be doing other things while waiting for an operation on the device to complete. There are multiple ways to handle this:

- In the **synchronous I/O model**, the application issues a request from a thread. The thread blocks until the operation is complete. The operating system creates the illusion that issuing the request to the device and receiving the result was just like any other operation that would proceed just on the CPU, but in reality, it may switch in other threads or processes to make use of the CPU resources and to allow other device requests to be issued to the device in parallel, originating from the same CPU.
- In the **asynchronous I/O (AIO) model**, the application can submit one or many requests from a thread. Submitting a request does not cause the thread to block, and instead the thread can proceed to do other computations and submit further requests to the device while the original request is in flight. The application is expected to process completions and organize logical computations itself without depending on threads to organize the use of data.

Asynchronous I/O can be considered “lower level” than synchronous I/O because it does not make use of a system-provided concept of threads to organize its computation. However, it is often more efficient to use AIO than synchronous I/O due the nondeterministic overhead of threads.

## The Linux AIO model
The Linux AIO model is used as follows:

1. Open an I/O context to submit and reap I/O requests from.
1. Create one or more request objects and set them up to represent the desired operation
1. Submit these requests to the I/O context, which will send them down to the device driver to process on the device
1. Reap completions from the I/O context in the form of event completion objects,
1. Return to step 2 as needed.

## I/O context
`io_context_t` is a pointer-sized opaque datatype that represents an “AIO context”. It can be safely passed around by value. Requests in the form of a `struct iocb` are submitted to an `io_context_t` and completions are read from the `io_context_t`. Internally, this structure contains a queue of completed requests. The length of the queue forms an upper bound on the number of concurrent requests which may be submitted to the `io_context_t`.

To create a new `io_context_t`, use the function

```c
int io_setup(int maxevents, io_context_t *ctxp);
```

Here, `ctxp` is the output and `maxevents` is the input. The function creates an `io_context_t` with an internal queue of length `maxevents`. To deallocate an `io_context_t`, use

```c
int io_destroy(io_context_t ctx);
```

There is a system-wide maximum number of allocated `io_context_t` objects, set at 65536.

An `io_context_t` object can be shared between threads, both for submission and completion. No guarantees are provided about ordering of submission and completion with respect to interaction from multiple threads. There may be performance implications from sharing `io_context_t` objects between threads.

## Submitting requests
`struct iocb` represents a single request for a read or write operation. The following struct shows a simplification on the struct definition; a full definition is found in `<libaio.h>` within the libaio source code.

```c
struct iocb {
    void *data;
    short aio_lio_opcode;
    int aio_fildes;

    union {
        struct {
            void *buf;
            unsigned long nbytes;
            long long offset;
        } c;
    } u;
};
```

The meaning of the fields is as follows: data is a pointer to a user-defined object used to represent the operation

- `aio_lio_opcode` is a flag indicate whether the operation is a read (`IO_CMD_PREAD`) or a write (`IO_CMD_PWRITE`) or one of the other supported operations
- `aio_fildes` is the fd of the file that the iocb reads or writes
- `buf` is the pointer to memory that is read or written
- `nbytes` is the length of the request
- `offset` is the initial offset of the read or write within the file

The convenience functions `io_prep_pread` and `io_prep_pwrite` can be used to initialize a `struct iocb`.
New operations are sent to the device with `io_submit`.

```c
int io_submit(io_context_t ctx, long nr, struct iocb *ios[]);
```

`io_submit` allows an array of pointers to `struct iocb`s to be submitted all at once. In this function call, `nr` is the length of the `ios` array. If multiple operations are sent in one array, then no ordering guarantees are given between the `iocb`s. Submitting in larger batches sometimes results in a performance improvement due to a reduction in CPU usage. A performance improvement also sometimes results from keeping many I/Os ‘in flight’ simultaneously.

If the submission includes too many iocbs such that the internal queue of the `io_context_t` would overfill on completion, then `io_submit` will return a non-zero number and set `errno` to `EAGAIN`.

When used under the right conditions, `io_submit` should not block. However, when used in certain ways, it may block, undermining the purpose of asynchronous I/O. If this is a problem for your application, be sure to use the `O_DIRECT` flag when opening a file, and operate on a raw block device. Work is ongoing to fix the problem.

## Processing results
Completions read from an `io_context_t` are of the type `struct io_event`, which contains the following relevant fields.

```c
struct io_event {
    void *data;
    struct iocb *obj;
    long long res;
};
```

Here, `data` is the same data pointer that was passed in with the `struct iocb`, and `obj` is the original `struct iocb`. `res` is the return value of the read or write.

Completions are reaped with `io_getevents`.

```c
int io_getevents(io_context_t ctx_id, long min_nr, long nr, struct io_event *events, struct timespec *timeout);
```

This function has a good number of parameters, so an explanation is in order:

- `ctx_id` is the `io_context_t` that is being reaped from.
- `min_nr` is the minimum number of `io_events` to return. `io_gevents` will block until there are `min_nr` completions to report, if this is not already the case when the function call is made.
- `nr` is the maximum number of completions to return. It is expected to be the length of the `events` array.
- `events` is an array of `io_events` into which the information about completions is written.
- `timeout` is the maximum time that a call to `io_getevents` may block until it will return. If `NULL` is passed, then `io_getevents` will block until `min_nr` completions are available.

The return value represents how many completions were reported, i.e. how much of events was written. The return value will be between 0 and `nr`. The return value may be lower than `min_nr` if the timeout expires; if the timeout is `NULL`, then the return value will be between `min_nr` and `nr`.

The parameters give a broad range of flexibility in how AIO can be used.

- `min_nr = 0` (or, equivalently, `timeout = 0`). This option forms a non-blocking polling technique: it will always return immediately, regardless of whether any completions are available. It makes sense to use `min_nr = 0` when calling `io_getevents` as part of a main run-loop of an application, on each iteration.
- `min_nr = 1`. This option blocks until a single completion is available. This parameter is the minimum value which will produce a blocking call, and therefore may be the best value for low latency operations for some users. When an application notices that an `eventfd` corresponding to an iocb is triggered (see the next section about `epoll`), then the application can call `io_getevents` on the corresponding `io_context_t` with a guarantee that no blocking will occur.
- `min_nr > 1`. This option waits for multiple completions to return, unless the timeout expires. Waiting for multiple completions may improve throughput due to reduced CPU usage, both due to fewer `io_getevents` calls and because if there is more space in the completion queue due to the removed completions, then a later `io_submit` call may have a larger granularity, as well as a reduced number of context switches back to the calling thread when the event is available. This option runs the risk of increasing the latency of operations, especially when the operation rate is lower.

Even if `min_nr = 0` or `1`, it is useful to make nr a bit bigger for performance reasons: more than one event may be already complete, and it could be processed without multiple calls to `io_getevents`. The only cost of a larger nr value library is that the user must allocate a larger array of events and be prepared to accept them.

## Use with epoll
Any `iocb` can be set to notify an `eventfd` on completion using the libaio function `io_set_eventfd`. The `eventfd` can be put in an `epoll` object. When the `eventfd` is triggered, then the `io_getevents` function can be called on the corresponding `io_context_t`.

There is no way to use this API to trigger an eventfd only when multiple operations are complete--the eventfd will always be triggered on the first operation. Consequently, as described in the previous section, it will often make sense to use `min_nr = 1` when using `io_getevents` after an `epoll_wait` call that indicates an `eventfd` involved in AIO.

## Performance considerations
- **Blocking during `io_submit` on ext4, on buffered operations, network access, pipes, etc.** Some operations are not well-represented by the AIO interface. With completely unsupported operations like buffered reads, operations on a socket or pipes, the entire operation will be performed during the io_submit syscall, with the completion available immediately for access with io_getevents. AIO access to a file on a filesystem like ext4 is partially supported: if a metadata read is required to look up the data block (ie if the metadata is not already in memory), then the io_submit call will block on the metadata read. Certain types of file-enlarging writes are completely unsupported and block for the entire duration of the operation.
- **CPU overhead.** When performing small operations on a high-performance device and targeting a very high operation rate from single CPU, a CPU bottleneck may result. This can be resolved by submitting and reaping AIO from multiple threads.
- **Lock contention when many CPUs or requests share an io_context_t.** There are several circumstances when the kernel datastructure corresponding to an io_context_t may be accessed from multiple CPUs. For example, multiple threads may submit and get events from the same io_context_t. Some devices may use a single interrupt line for all completions. This can cause the lock to be bounced around between cores or the lock to be heavily contended, resulting in higher CPU usage and potentially lower throughput. One solution is to shard into multiple io_context_t objects, for example by thread and a hash of the address.
- **Ensuring sufficient parallelism.** Some devices require many concurrent operations to reach peak performance. This means making sure that there are several operations ‘in flight’ simultaneously. On some high-performance storage devices, when operations are small, tens or hundreds must be submitted in parallel in order to achieve maximum throughput. For disk drives, performance may improve with greater parallelism if the elevator scheduler can make better decisions with more operations simultaneously in flight, but the effect is expected to be small in many situations.

## Alternatives to Linux AIO
- **Thread pool of synchronous I/O threads.** This can work for many use cases, and it may be easier to program with. Unlike with AIO, all functions can be parallelized via a thread pool. Some users find that a thread pool does not work well due to the overhead of threads in terms of CPU and memory bandwidth usage from context switching. This comes up as an especially big problem with small random reads on high-performance storage devices.
- **POSIX AIO.** Another asynchronous I/O interface is POSIX AIO. It is implemented as part of glibc. However, the glibc implementation uses a thread pool internally. For cases where this is acceptable, it might be better to use your own thread pool instead. Joel Becker implemented [a version](https://oss.oracle.com/projects/libaio-oracle/files/) of POSIX AIO based on the Linux AIO mechanism described above. IBM DeveloperWorks has [a good introduction](http://www.ibm.com/developerworks/linux/library/l-async/index.html) to POSIX AIO.
- **epoll.** Linux has limited support for using epoll as a mechanism for asynchronous I/O. For reads to a file opened in buffered mode (that is, without O_DIRECT), if the file is opened as O_NONBLOCK, then a read will return EAGAIN until the relevant part is in memory. Writes to a buffered file are usually immediate, as they are written out with another writeback thread. However, these mechanisms don’t give the level of control over I/O that direct I/O gives.

## Sample code
Below is some example code which uses Linux AIO. I wrote it at Google, so it uses the [Google glog logging library](https://github.com/google/glog) and the [Google gflags command-line flags library](http://gflags.github.io/gflags/), as well as a loose interpretation of [Google’s C++ coding conventions](https://google.github.io/styleguide/cppguide.html). When compiling it with gcc, pass `-laio` to dynamically link with libaio. (It isn’t included in glibc, so it must be explicitly included.)

```c
// Code written by Daniel Ehrenberg, released into the public domain

#include <fcntl.h>
#include <gflags/gflags.h>
#include <glog/logging.h>
#include <libaio.h>
#include <stdlib.h>
#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>

DEFINE_string(path, "/tmp/testfile", "Path to the file to manipulate");
DEFINE_int32(file_size, 1000, "Length of file in 4k blocks");
DEFINE_int32(concurrent_requests, 100, "Number of concurrent requests");
DEFINE_int32(min_nr, 1, "min_nr");
DEFINE_int32(max_nr, 1, "max_nr");

// The size of operation that will occur on the device
static const int kPageSize = 4096;

class AIORequest {
 public:
  int* buffer_;

  virtual void Complete(int res) = 0;

  AIORequest() {
    int ret = posix_memalign(reinterpret_cast<void**>(&buffer_),
                             kPageSize, kPageSize);
    CHECK_EQ(ret, 0);
  }

  virtual ~AIORequest() {
    free(buffer_);
  }
};

class Adder {
 public:
  virtual void Add(int amount) = 0;

  virtual ~Adder() { };
};

class AIOReadRequest : public AIORequest {
 private:
  Adder* adder_;

 public:
  AIOReadRequest(Adder* adder) : AIORequest(), adder_(adder) { }

  virtual void Complete(int res) {
    CHECK_EQ(res, kPageSize) << "Read incomplete or error " << res;
    int value = buffer_[0];
    LOG(INFO) << "Read of " << value << " completed";
    adder_->Add(value);
  }
};

class AIOWriteRequest : public AIORequest {
 private:
  int value_;

 public:
  AIOWriteRequest(int value) : AIORequest(), value_(value) {
    buffer_[0] = value;
  }

  virtual void Complete(int res) {
    CHECK_EQ(res, kPageSize) << "Write incomplete or error " << res;
    LOG(INFO) << "Write of " << value_ << " completed";
  }
};

class AIOAdder : public Adder {
 public:
  int fd_;
  io_context_t ioctx_;
  int counter_;
  int reap_counter_;
  int sum_;
  int length_;

  AIOAdder(int length)
      : ioctx_(0), counter_(0), reap_counter_(0), sum_(0), length_(length) { }

  void Init() {
    LOG(INFO) << "Opening file";
    fd_ = open(FLAGS_path.c_str(), O_RDWR | O_DIRECT | O_CREAT, 0644);
    PCHECK(fd_ >= 0) << "Error opening file";
    LOG(INFO) << "Allocating enough space for the sum";
    PCHECK(fallocate(fd_, 0, 0, kPageSize * length_) >= 0) << "Error in fallocate";
    LOG(INFO) << "Setting up the io context";
    PCHECK(io_setup(100, &ioctx_) >= 0) << "Error in io_setup";
  }

  virtual void Add(int amount) {
    sum_ += amount;
    LOG(INFO) << "Adding " << amount << " for a total of " << sum_;
  }

  void SubmitWrite() {
    LOG(INFO) << "Submitting a write to " << counter_;
    struct iocb iocb;
    struct iocb* iocbs = &iocb;
    AIORequest *req = new AIOWriteRequest(counter_);
    io_prep_pwrite(&iocb, fd_, req->buffer_, kPageSize, counter_ * kPageSize);
    iocb.data = req;
    int res = io_submit(ioctx_, 1, &iocbs);
    CHECK_EQ(res, 1);
  }

  void WriteFile() {
    reap_counter_ = 0;
    for (counter_ = 0; counter_ < length_; counter_++) {
      SubmitWrite();
      Reap();
    }
    ReapRemaining();
  }

  void SubmitRead() {
    LOG(INFO) << "Submitting a read from " << counter_;
    struct iocb iocb;
    struct iocb* iocbs = &iocb;
    AIORequest *req = new AIOReadRequest(this);
    io_prep_pread(&iocb, fd_, req->buffer_, kPageSize, counter_ * kPageSize);
    iocb.data = req;
    int res = io_submit(ioctx_, 1, &iocbs);
    CHECK_EQ(res, 1);
  }

  void ReadFile() {
    reap_counter_ = 0;
    for (counter_ = 0; counter_ < length_; counter_++) {
        SubmitRead();
        Reap();
    }
    ReapRemaining();
  }

  int DoReap(int min_nr) {
    LOG(INFO) << "Reaping between " << min_nr << " and "
              << FLAGS_max_nr << " io_events";
    struct io_event* events = new io_event[FLAGS_max_nr];
    struct timespec timeout;
    timeout.tv_sec = 0;
    timeout.tv_nsec = 100000000;
    int num_events;
    LOG(INFO) << "Calling io_getevents";
    num_events = io_getevents(ioctx_, min_nr, FLAGS_max_nr, events,
                              &timeout);
    LOG(INFO) << "Calling completion function on results";
    for (int i = 0; i < num_events; i++) {
      struct io_event event = events[i];
      AIORequest* req = static_cast<AIORequest*>(event.data);
      req->Complete(event.res);
      delete req;
    }
    delete events;
    
LOG(INFO) << "Reaped " << num_events << " io_events";
    reap_counter_ += num_events;
    return num_events;
  }

  void Reap() {
    if (counter_ >= FLAGS_min_nr) {
      DoReap(FLAGS_min_nr);
    }
  }

  void ReapRemaining() {
    while (reap_counter_ < length_) {
      DoReap(1);
    }
  }

  ~AIOAdder() {
    LOG(INFO) << "Closing AIO context and file";
    io_destroy(ioctx_);
    close(fd_);
  }

  int Sum() {
    LOG(INFO) << "Writing consecutive integers to file";
    WriteFile();
    LOG(INFO) << "Reading consecutive integers from file";
    ReadFile();
    return sum_;
  }
};

int main(int argc, char* argv[]) {
  google::ParseCommandLineFlags(&argc, &argv, true);
  AIOAdder adder(FLAGS_file_size);
  adder.Init();
  int sum = adder.Sum();
  int expected = (FLAGS_file_size * (FLAGS_file_size - 1)) / 2;
  LOG(INFO) << "AIO is complete";
  CHECK_EQ(sum, expected) << "Expected " << expected << " Got " << sum;
  printf("Successfully calculated that the sum of integers from 0"
         " to %d is %d\n", FLAGS_file_size - 1, sum);
  return 0;
}
```
