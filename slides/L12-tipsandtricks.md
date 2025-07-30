# Effective MPI: Tips and Tricks for Robust Parallel Programming

Message Passing Interface (MPI) is the cornerstone of high-performance computing, but writing programs that are correct, efficient, portable, and easy to maintain can be a significant challenge. This article, adapted from a lecture on advanced MPI, distills years of experience into practical tips and tricks to avoid common pitfalls and improve your parallel programming.

## Aims of a Good MPI Program

Before diving into specific problems, it's essential to define our goals. We aim to write MPI programs that are:

  * **Correct**: They produce the right results without deadlocks or race conditions.
  * **Portable**: They can run on different systems, with different compilers and MPI libraries, without modification.
  * **Efficient**: They scale well and make good use of the underlying parallel hardware.
  * **Maintainable**: The code is readable, and debugging is straightforward.

With these aims in mind, let's explore the most common problems developers face and how to solve them.

-----

## Common Problems and Solutions

### 1\. Assuming `MPI_Send` is Asynchronous

A frequent and dangerous assumption is that `MPI_Send` behaves asynchronously—that it returns immediately without waiting for the corresponding receive to be posted. This can lead to code that works on one machine but deadlocks on another.

The MPI standard does not guarantee that `MPI_Send` is asynchronous. For small messages, MPI implementations often buffer the message and return immediately, giving the *illusion* of asynchronous behavior. However, for larger messages, the send will often block until a matching `MPI_Recv` is initiated. The threshold between buffered and blocking behavior varies between MPI implementations, making any code that relies on this a portability nightmare.

**Solution:**

  * **Program Defensively**: Always write your code as if `MPI_Send` is synchronous. Ensure a matching receive is posted before, or concurrently with, the send to prevent deadlocks.
  * **Test for Correctness**: A robust test for correctness is to replace all `MPI_Send` calls in your code with `MPI_Ssend` (synchronous send). If the code still runs without deadlocking, it is correct.
  * **Avoid `MPI_Bsend` (Buffered Send)**: While `MPI_Bsend` explicitly offers buffered, asynchronous sends, it requires you to manage the buffer memory yourself. The default buffer size may be zero, and it's a notoriously tricky routine to use correctly. It's often better to use non-blocking communication instead.

### 2\. Handling Data Precision and Portability

Code that relies on specific data sizes (e.g., assuming an `int` is always 4 bytes) or uses compiler-specific flags to change precision (like `-convert-floats-to-doubles`) is inherently non-portable. When you change from `float` to `double` in your variable declarations, you must also change the corresponding MPI types from `MPI_FLOAT` to `MPI_DOUBLE` in your communication calls.

**Solution: Use an Include File for Precision**

The easiest way to manage precision is to abstract it away using a header file or a module. This allows you to change the precision of your entire application by editing just one or two lines.

**For C:**
Create a header file, e.g., `precision.h`:

```c
typedef float RealNumber;
#define MPI_REALNUMBER MPI_FLOAT
```

In your main code, use these definitions:

```c
#include "precision.h"

RealNumber x;
MPI_Send(&x, 1, MPI_REALNUMBER, ...);
```

To switch to double precision, you only need to edit `precision.h`:

```c
typedef double RealNumber;
#define MPI_REALNUMBER MPI_DOUBLE
```

**For Fortran:**
Create a module, e.g., `precision_mod.f90`:

```fortran
module precision
  implicit none
  integer, parameter :: REALNUMBER = kind(1.0e0)  ! Single precision
  integer, parameter :: MPI_REALNUMBER = MPI_REAL
end module precision
```

In your subroutines, use the module:

```fortran
subroutine my_sub
  use precision
  implicit none
  real(kind=REALNUMBER) :: x

  call MPI_SEND(x, 1, MPI_REALNUMBER, ...)
end subroutine my_sub
```

To switch to double precision, you only need to edit the module:

```fortran
integer, parameter :: REALNUMBER = kind(1.0d0)  ! Double precision
integer, parameter :: MPI_REALNUMBER = MPI_DOUBLE_PRECISION
```

### 3\. Improving Code Readability by Separating Communications

Sprinkling MPI calls throughout your application logic makes the code hard to read, debug, and maintain. A much better approach is to abstract all parallel operations into a dedicated communications library or file. This keeps your main application code clean and independent of the specific MPI implementation.

**Example: A Global Sum**

**The "Ugly" Way (in the main logic):**

```c
// Globally sum the double precision rainfall value
double partial_rainfall = ...;
double total_rainfall;
MPI_Allreduce(&partial_rainfall, &total_rainfall, 1, MPI_DOUBLE, MPI_SUM, MPI_COMM_WORLD);
```

**The "Nicer" Way (using a communications library):**
In your main logic:

```c
// Globally sum the double precision rainfall value
rainfall = parallel_sum_double(rainfall);
```

In your parallel communications library (`libpar.c`):

```c
double parallel_sum_double(double dval) {
  double dsum;
  MPI_Allreduce(&dval, &dsum, 1, MPI_DOUBLE, MPI_SUM, MPI_COMM_WORLD);
  return dsum;
}
```

**Advantages of a Communications Library:**

  * **Maintain a Serial Version**: You can create a "dummy" serial library with the same function names that does nothing. This allows you to compile and run a true serial version of your code from the same source files, which is invaluable for debugging.
  * **Easier Debugging**: All parallel logic is centralized. If you need to add print statements or timing calls to your global sums, you only need to do it in one place.
  * **Simplified Porting**: If you need to port to another communication library (e.g., Cray's SHMEM), you only need to rewrite your small communications library, not the entire application.
  * **Targeted Optimization**: You can have different implementations of a communication routine and choose the fastest one for a specific machine or MPI library.

### 4\. Calling Collectives Correctly

Collective operations like `MPI_Bcast` or `MPI_Allreduce` **must be called by all processes** in the communicator. A common beginner mistake is to place a collective call inside a conditional statement.

**Incorrect:**

```c
if (rank == 0) {
  MPI_Bcast(buffer, count, MPI_INT, 0, MPI_COMM_WORLD);
}
```

This will cause all processes other than rank 0 to hang, as they will be waiting for a broadcast that rank 0 never initiated for them. An `MPI_Allreduce` called this way would deadlock indefinitely.

**Tip: Prefer `MPI_Allreduce` over Reduce + Broadcast**
Sometimes, a value needs to be calculated globally (e.g., a sum) and then made available to all processes for an error check or subsequent calculation. While you could use `MPI_Reduce` to send the result to a root process and then `MPI_Bcast` it back out, it's simpler and often more performant to just use a single `MPI_Allreduce` call.

### 5\. Avoiding Lots of Small Messages

Every MPI message, no matter how small, incurs a latency cost—an overhead for initiating the communication. This latency can be thousands of times more expensive than a floating-point operation. Sending many small messages in a loop is one of the most common performance killers in MPI.

**Inefficient:**

```c
// Sending N integers one at a time
for (j = 0; j < N; j++) {
  MPI_Send(&x[0][j], 1, MPI_INT, dest, ...);
}
```

**Efficient:**

```c
// Sending N integers in a single message
MPI_Send(&x[0][0], N, MPI_INT, dest, ...);
```

If your data is non-contiguous (e.g., a column in a C 2D array or a row in Fortran), use a derived datatype like `MPI_Type_vector` to describe the memory layout and send it as a single message.

### 6\. Debugging Parallel Programs

Debugging is hard, and parallel debugging is even harder. The biggest mistake is to assume every bug is a parallel bug.

**A Step-by-Step Debugging Strategy:**

1.  **Don't Assume It's a Parallel Bug**: The first step is always to check the serial version of your code. If you've separated your communications into a library, this is easy.
2.  **Run on P=1**: Run the parallel version on a single process. This can catch issues that only appear when the MPI library is initialized.
3.  **Scale Up Slowly**: Run on a small number of processes (2, 4, etc.). A bug will typically manifest on a handful of processes, which is much easier to debug than on hundreds.
4.  **Write Verbose, Annotated Output**: Don't write silent programs\! Your code should announce what it's doing: "Running on 6 processes," "Reading input file," "Starting iteration 100." This helps you know how far the program got before it crashed. Never write raw numbers to the screen; always annotate them (e.g., `printf("rank, j, x: %d %d %f\n", rank, j, x);`).
5.  **Find the Bug, Don't Hide It**: If you change something (like the compiler or add a print statement) and the bug "goes away," it's not fixed. The bug is still there, just hidden. You **must** find the root cause, or it will return to bite you later.

### 7\. Fortran Array Syntax and Non-Blocking Calls

Fortran's powerful array syntax is convenient, but it can hide a subtle and dangerous bug when used with non-blocking MPI calls like `MPI_Issend`.

Consider this code:

```fortran
call MPI_Issend(array(1:m, 1), m, MPI_REAL, ...)
```

To handle the non-contiguous array section, the compiler will likely generate temporary code behind the scenes that looks like this:

```fortran
! Compiler's hidden actions
allocate(temp_buffer(m))
temp_buffer(1:m) = array(1:m, 1)  ! Copy data into a contiguous buffer
call MPI_Issend(temp_buffer, m, MPI_REAL, ...)
deallocate(temp_buffer) ! <-- DANGER!
```

The problem is that the `deallocate` may happen *before* the non-blocking send has actually completed, meaning MPI might try to read from deallocated memory.

**Solution:**

  * For contiguous sections, pass the starting element: `call MPI_Issend(array(1,1), m, ...)`
  * For non-contiguous sections, use a derived datatype (like a vector type) instead of Fortran array syntax with non-blocking calls. This is the safest and most portable approach. Modern Fortran 2008 interfaces may handle this correctly, but it's best to be cautious.

By keeping these tips in mind, you can avoid common pitfalls and write MPI code that is not only fast but also correct, portable, and far easier to debug and maintain.
