+++
title = "Introduction Note: MPI"
date = "2022-03-01 01:04:44"
authors = ["茨月"]
+++

In Tuesday’s Introduction to High-Performance Computing class, MPI was mentioned, and it was also used in a small assignment. However, I didn’t fully understand the code. So, I found an [MPI tutorial](https://mpitutorial.com/tutorials/mpi-introduction/zh_cn/). The Chinese translation is of good quality, but it’s still necessary to refer to the English version when needed.

<!-- more -->

## Basic Facts

- MPI stands for Message Passing Interface.

- MPI is a standard, and there are many implementations.
    - These include but are not limited to OpenMPI (used on the conv cluster) and MPICH (used in the tutorial).

- MPI provides parallel function libraries for a series of programming languages, including Fortran77, C, Fortran90, and C++.

- MPI is process-level parallelism, where memory is not shared between processes, unlike thread-level parallelism.

## Quick Usage Guide

### Basic Requirements

- The MPI program should call the `MPI_Init(int* argc, char*** argv)` function at the entry point. In practice, both parameters can be set to `NULL`.

- The MPI program should call the `MPI_Finalize()` function before exiting.

### Point-to-Point Communication Between Processes

- Use `MPI_Comm_size(MPI_COMM_WORLD, &world_size)` to get the total number of processes, which is stored in `world_size`.

- Use `MPI_Comm_rank(MPI_COMM_WORLD, &world_rank)` to get the process’s own rank, which is stored in `world_rank`.

- Use `MPI_Send(const void *buf, int count, MPI_Datatype datatype, int dest, int tag, MPI_Comm comm)` to send data from one process to another.
    - `buf` is the send buffer.
    - `count` is the number of data items to send.
    - `datatype` is the data type, an internal MPI enumeration that corresponds to native C types.
    - `dest` is the rank of the destination process.
    - `tag` is the message tag; only messages with the corresponding tag will be received.
    - `comm` is the communicator.

- Use `MPI_Recv(void *buf, int count, MPI_Datatype datatype, int source, int tag, MPI_Comm comm, MPI_Status *status)` to receive data from another process.
    - `buf` is the receive buffer.
    - `count` is the maximum number of data items to receive.
        - It can receive up to this number of items; exceeding it will cause an error.
    - `datatype` is the data type.
    - `source` is the rank of the source process.
        - `MPI_ANY_SOURCE` can be used to receive messages from any source.
    - `tag` is the message tag; only messages with the corresponding tag will be received.
        - `MPI_ANY_TAG` can be used to receive messages with any tag.
    - `comm` is the communicator.
    - `status` is a pointer to an `MPI_Status` structure, which stores additional information about the received message.
        - `MPI_STATUS_IGNORE` can be used to discard this information.

- `MPI_Send` and `MPI_Recv` are blocking; they will not proceed until the send/receive operation is complete.

- To send custom types:
    - Cast the buffer to `void *`.
    - Set `count` to the number of items multiplied by `sizeof(MyType)`.
    - Set `datatype` to `MPI_BYTE`.

- The `MPI_Status` structure records the following:
    - The rank of the sending process.
    - The message tag.
    - The length of the message (the number of elements can be determined after specifying the type).

- Use `MPI_Get_count(MPI_Status* status, MPI_Datatype datatype, int* count)` to get the actual number of elements in the message.

- Use `MPI_Probe(int source, int tag, MPI_Comm comm, MPI_Status* status)` to obtain `MPI_Status` without receiving the message.
    - If the exact number of elements to receive is unknown, you can first use `MPI_Probe` to get `status`, use `MPI_Get_count` to calculate the number of elements, and then use `MPI_Recv`. At this point, you can use `MPI_STATUS_IGNORE`.

### Multi-Process Synchronization

- Use `MPI_Barrier(MPI_Comm communicator)` to synchronize processes.
    - All processes must reach this point before proceeding; otherwise, the program will hang.

- Use `MPI_Bcast(const void *buf, int count, MPI_Datatype datatype, int root, MPI_Comm comm)` to broadcast/receive broadcast data.
    - If the process’s rank is `root`, it sends `buf`; otherwise, it receives.
    - `MPI_Bcast` uses a tree-based broadcast algorithm for efficient broadcasting.
        - Simple tests show that on a single machine with multiple cores (i.e., no network communication), the efficiency of `MPI_Bcast` is approximately $\log N$ times that of a naive linear implementation, i.e., 2 threads perform the same, 4 threads perform 2x, 8 threads perform 3x...
        - Does its broadcasting behavior depend on the network topology? In other words, for heterogeneous clusters, will it find a "better" broadcast path?

- Use `MPI_Scatter(void* send_data, int send_count, MPI_Datatype send_datatype, void* recv_data, int recv_count, MPI_Datatype recv_datatype, int root, MPI_Comm communicator)` to split data in a buffer and send segments to other processes.
    - `send_data` is the send buffer.
    - `send_count` is the number of elements to send.
    - `send_datatype` is the type of elements to send.
    - `recv_data` is the receive buffer.
    - `recv_count` is the number of elements to receive.
        - Generally, it should be a divisor of `send_count`.
    - `recv_datatype` is the type of elements to receive.
        - It’s usually the same as the send type.
    - `root` is the sending process.
    - `communicator` is the communicator.

<div class="side-by-side-container"><img alt="Bcast vs Scatter" id="should-invert" src="/images/mpi-reference/broadcastvsscatter.webp"></div>

- Use `MPI_Gather(void* send_data, int send_count, MPI_Datatype send_datatype, void* recv_data, int recv_count, MPI_Datatype recv_datatype, int root, MPI_Comm communicator)` to gather data from all processes into a large buffer in the root process, the opposite of `MPI_Scatter`. Since the function signature is the same, the parameters are not repeated here.

<div class="side-by-side-container"><img alt="Gather" id="should-invert" src="/images/mpi-reference/gather.webp"/></div>

- A common workflow is as follows:
    - Process 0 is the root process, and the others are worker processes.
    - The root prepares the data and scatters it to the worker processes, which perform the operations.
    - Use `Barrier` to ensure all worker operations are complete.
    - The root gathers all the data back.
    - This feels similar to Map, XD.

- Use `MPI_Allgather(void* send_data, int send_count, MPI_Datatype send_datatype, void* recv_data, int recv_count, MPI_Datatype recv_datatype, MPI_Comm communicator)` to perform a gather followed by a broadcast, so that all processes have the same copy of the data in their buffers. The function signature is the same as `MPI_Gather`, except there is no `root` (it’s no longer needed), so it’s not repeated here.

<div class="side-by-side-container"><img alt="Allgather" id="should-invert" src="/images/mpi-reference/allgather.webp"/></div>

- Use `MPI_Reduce(void* send_data, void* recv_data, int count, MPI_Datatype datatype, MPI_Op op, int root, MPI_Comm communicator)` to perform a reduce operation on data from multiple processes.
    - `send_data` is the send buffer for each process’s data to be reduced.
    - `recv_data` is the receive buffer in the root process for the reduced result.
    - `count` is the number of elements in the buffer.
    - `datatype` is the element type.
    - `op` is the predefined reduce operation in MPI, including max/min, sum/product, bitwise AND/OR, and the rank of the max/min value.
    - `root` is the root process that receives the reduced result.
    - `communicator` is the communicator.

<div class="side-by-side-container"><img alt="count = 1 的 reduce" id="should-invert" src="/images/mpi-reference/mpi_reduce_1.webp"/></div>
<div class="side-by-side-container"><img alt="count = 2 的 reduce" id="should-invert" src="/images/mpi-reference/mpi_reduce_2.webp"/></div>

- `MPI_Allreduce` is to `MPI_reduce` what `Allgather` is to `gather`. It sends the reduced data to all processes, so the signature is `MPI_Allreduce(void* send_data, void* recv_data, int count, MPI_Datatype datatype, MPI_Op op, MPI_Comm communicator)`, with no `root`, and the rest remains the same.

<div class="side-by-side-container"><img alt="Allreduce" id="should-invert" src="/images/mpi-reference/mpi_allreduce_1.webp"/></div>

### Communicators and Groups

- A "group" can be considered a collection of processes.

- A communicator corresponds to a group and is used for communication between processes in the group.
    - `MPI_COMM_WORLD` is the global communicator that includes all processes.
    - Use `MPI_Comm_Rank(MPI_Comm comm, int *rank)` to get the rank of the process within the communicator.
    - Use `MPI_Comm_Size(MPI_Comm comm, int *size)` to get the number of processes in the communicator.

- Use `MPI_Comm_split(MPI_Comm comm, int color, int key, MPI_Comm* newcomm)` to split the old communicator and create a new one.
    - `comm` is the old communicator.
    - Processes with the same `color` will be grouped into the same new communicator.
    - The order of `rank` will be used to determine the new rank within the group.
    - `new_comm` is the newly created communicator.

For example, run the following code with 16 processes (only the core part is retained):

```c
int world_rank, world_size;
MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
MPI_Comm_size(MPI_COMM_WORLD, &world_size);

int color = world_rank / 4;

MPI_Comm row_comm;
MPI_Comm_split(MPI_COMM_WORLD, color, world_rank, &row_comm);

int row_rank, row_size;
MPI_Comm_rank(row_comm, &row_rank);
MPI_Comm_size(row_comm, &row_size);

printf("World rank & size: %d / %d\t", world_rank, world_size);
printf("Row rank & size: %d / %d\n", row_rank, row_size);
```

Originally, the process with `(world_rank, 16)` will become `(world_rank / 4, 16)` in the new communicator.

<div class="side-by-side-container"><img alt="MPI_Comm_split" id="should-invert" src="/images/mpi-reference/comm_split.webp"/></div>

If `world_rank / 4` is changed to `world_rank % 4`, the division will change from horizontal to vertical, with a similar principle.

- Use `MPI_Comm_create(MPI_Comm comm, MPI_Group group, MPI_Comm *newcomm)` and `MPI_Comm_create_group(MPI_Comm comm, MPI_Group group, int tag, MPI_Comm *newcomm)` to create a new communicator from a group.
    - I didn’t fully understand the difference.

- `MPI_Group` can perform set operations like intersection and union.

Honestly, this part of the tutorial is too brief, and I didn’t fully understand it.

According to students who took the course last year, when using MPI in class, you generally don’t need to create communicators by splitting groups; you can just use `MPI_COMM_WORLD` directly. So, I won’t delve deeper here.

### Miscellaneous

- Use `MPI_Wtime()` for timing.
    - It returns the number of seconds since 1970-1-1, i.e., the UNIX timestamp.
