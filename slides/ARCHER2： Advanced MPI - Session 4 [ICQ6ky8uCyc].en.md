### Introduction to the Halo Swapping Benchmark

I realized that the halo swapping stuff actually requires more of an introduction. I will go to the GitHub page to explain.

A while back, based on information from a previous lecture, people asked me what happens in practice. So, what I did is I wrote a simple benchmark. I set the processes up into a 3D grid, and each of them sends data up, down, left, right, forward, and backward. I thought of every possible way you could do that.

### Methods for Halo Swapping

I tested about nine different ways of doing halo swapping.

First, I did pairwise swapping using send-receive. Basically, you do a send-receive in every direction, so that would be six send-receives for up, down, left, right, forward, and backward. You have to use non-blocking communication because if you issue the receive first, you may never get to the send, and if you issue the send first, you may never get to the receive.

A trick you can use is a two-pass method, often called a checkerboard or red-black algorithm. You split your grid into two sets, odd and even processes. In the first part, the odd processes send while the even ones receive, and then they switch.

I also tried non-blocking sends and receives in various combinations: a non-blocking send, a receive, and then a wait; a non-blocking receive, a send, and a wait; or a non-blocking send and a non-blocking receive followed by a wait.

The method I predicted would be the best was to issue all communications at once: you issue six I-receives, six I-sends, and then just do a single wait-all. I also did the same thing with persistent communications.

Finally, I tested neighborhood collectives. These allow you to do halo swapping with a single call. Unlike a routine like `MPI_Alltoall` where every process communicates with every other one, a neighborhood collective like `MPI_Neighbor_alltoall` lets each process communicate only with its defined neighbors—in this case, the six nearest neighbors in the grid.

### Running the Benchmark Code

The code is written in Fortran. There's a driver routine that does something fairly simple. It takes the number of repetitions, send and receive buffers, the number of elements, and a Cartesian communicator which arranges the processes in a 3D grid. The code loops over the dimensions (X, Y, Z) and directions (up, down) and performs the send-receive calls.

The whole benchmark, called `halo_bench`, is run three times. This is because there can be a warm-up period on the system, and the first time you run, you may not get reliable results. I run it three times, so you should look at the final run for the most accurate answers.

By default, the benchmark transmits 10,000 doubles, which is about 80 kilobytes, and repeats this a thousand times for each method. If you make the buffer size bigger, you should make the number of repetitions smaller to keep the runtime sensible, around a second or so.

To run it, you can download the code from the GitHub repository. There's a makefile provided. By default, the submission script runs on eight nodes with 128 tasks per node, for a total of 1,024 processes in a 3D grid.

### Analysis of Results

Looking at the output from a run, you can see a clear hierarchy of which methods are best. The results print out the time taken and the bandwidth for each method.

When running with nodes fully packed with processes, methods that impose a strict order on the communications (e.g., completing all X-dimension swaps, then all Y, then all Z) are much slower, achieving around 670 megabytes per second. This is because you don't know the order in which messages will arrive; some neighbors might be on the same node (fast communication) while others are on different nodes (slower communication), and enforcing a strict sequence creates bottlenecks.

In contrast, the methods that issue all communications non-blockingly at once and then wait for them all to complete are significantly better. The non-blocking I-send/I-receive with wait-all, the persistent communication version, and the neighborhood collective all perform almost identically, achieving over a gigabyte per second. This is a significant improvement.

When we change the setup to run only one task per node, ensuring all communication goes over the network, the results change. Because every communication path is now more symmetrical and takes roughly the same amount of time, the performance difference between the ordered and non-blocking methods shrinks considerably. The overall bandwidth also increases dramatically to around 5 gigabytes per second because there's no network contention on the node.

### Advanced Collectives and Data Types

This leads to the topic of advanced collectives. At first sight, it looks like you can't use routines like `MPI_Scatter` and `MPI_Gather` for distributing a 2D matrix that is decomposed across both its dimensions. However, it turns out you can, but you have to play around with MPI data types.

The trick is that when you send multiple custom data types (like a vector with gaps in it), MPI's default definition of the object's size isn't always useful for calculating the displacement to the next element. You have to use a rather nasty but powerful function called `MPI_Type_create_resized` to tell MPI explicitly how to measure the extent of your custom data type.

While it's a bit technical, it is worth doing because using optimized collectives will be so much more efficient than trying to implement the communication pattern by hand with point-to-point messages.
