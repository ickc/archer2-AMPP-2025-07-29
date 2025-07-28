<img src="./images/Archer2_logo.png" width="355" height="100"
align="left"> <img src="./images/epcc_logo.jpg" align="right"
width="133" height="100">

<br /><br /><br /><br /><br />

# ARCHER2 Advanced MPI course (July 2025)

[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]

<h3>David Henty EPCC: 29 - 30 July 2025 10:00 - 16:00 BST, online</h3>

This course is aimed at programmers seeking to deepen their
understanding of MPI and explore some of its more recent and advanced
features. We cover topics including exploiting shared-memory access
from MPI programs, communicator management and advanced use of
collectives. We also look at performance aspects such as which MPI
routines to use for scalability, MPI internal implementation issues
and overlapping communication and calculation.  Intended learning
outcomes

*  Understanding of how internal MPI implementation details affect performance
*  Techniques for overlapping communications and calculation
*  Understanding of best practice for MPI+OpenMP programming
*  Knowledge of MPI memory models for RMA operations

<h3>Prerequisites</h3>

Attendees should be familiar with MPI programming in C, C++ or
Fortran, e.g. have attended the ARCHER2 MPI course.

<h3>Requirements</h3>

Participants must bring a laptop with a Mac, Linux, or Windows
operating system (not a tablet, Chromebook, etc.) that they have
administrative privileges on.

They are also required to abide by the [ARCHER2 Code of Conduct](https://www.archer2.ac.uk/about/policies/code-of-conduct.html).

<h3>Timetable (all times are in British Summer Time)</h3>

<p><blockquote>Although the start and end times will be as indicated below, this is a draft timetable based on
a previous run of the course and the details may change for this run.
</blockquote></p>

<p><blockquote>Unless otherwise indicated all material is Copyright
&copy; EPCC, The University of Edinburgh, and is only made available
for private study. </blockquote></p>

<h4>Day 1: Tuesday 29th July</h4>

 *   10:00 - 10:15 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/L00-ARCHER2-Intro.pdf">ARCHER2 training</a>
 *   10:15 - 11:15 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/L12-tipsandtricks.pdf">Tips and Tricks</a>
 *   11:15 - 11:45 Coffee Break
 *   11:45 - 12:30 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/MPI-Internals.pdf">MPI Internals</a>
 *   12:30 - 13:30 Lunch
 *   13:30 - 14:00 Practical: Point-to-point performance
 *   14:00 - 14:45 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/MPI-Optimisation-ARCHER2.pdf">MPI Optimisations</a>
 *   14:45 - 15:00 Tea Brea
 *   15:00 - 15:30 Practical: Halo Swapping
  *  15:30 - 16:00  <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/AMPP-Advanced-Collectives.pdf">Advanced Collectives<a>
 *   16:00 CLOSE

<h4>Day 2: Wednesday 30th July</h4>

 *   10:00 - 10:30 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/E01-traffic.pdf">Traffic Modelling Example<a>
  *  10:30 - 11:15 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/L06-MPIandOpenMP.pdf">MPI + OpenMP (i)<a>
 *   11:15 - 11:30 Coffee Break
 *   11:30 - 12:00 Practical: Traffic Model
 *   12:00 - 12:30 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/L06-MPIandOpenMP.pdf">MPI + OpenMP (ii)<a>
 *   12:30 - 13:30 Lunch
 *   13:30 - 14:00 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/IntroRMA.pdf">RMA Access in MPI</a>
 *   14:00 - 14:45 <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/slides/SharedMemoryRMA.pdf">New MPI shared-memory model</a>
 *   15:30 - 16:00 Practical: Traffic Model
 *   16:00 CLOSE

<h3>Exercise Material</h3>

<p><blockquote>Unless otherwise indicated all material is Copyright &copy; EPCC, The University of Edinburgh, and is only made available for private study. </blockquote></p>

<h4>Day 1</h4>

SLURM batch scripts are set to run in the short queue and should work any time. However, on days when the course is running, we have
special reserved queues to guarantee fast turnaround.

The reserved queue for today is called `ta206_1555696`. To use this queue, change the `--qos` and `--reservation` lines to:
````
#SBATCH --qos=reservation
#SBATCH --reservation=ta206_1555696
````

 * <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/exercises/ARCHER2-pingpong.pdf">Ping-pong exercise sheet</a>
 * <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/exercises/pingpong.tar">Ping-pong source code</a>
   
 * Description of 3D halo-swapping benchmark is in this <a href="https://github.com/davidhenty/halobench/">README</a>
 * Download the code directly to ARCHER2 using: `git clone https://github.com/davidhenty/halobench`
   - compile with `make -f Makefile-archer2`
   - submit with `sbatch archer2.job`
 * Other things you could do with the halo swapping benchmark:
   - change the buffer size to be very small ( a few tens of bytes) or very large (bigger than the eager limit) to see if that affects the results;
   - run on different numbers of nodes.
   - change the underlying network protocol from OFI to UCX - see https://docs.archer2.ac.uk/user-guide/dev-environment/#switching-to-alternative-ucx-mpi-implementation
 * Note that you will need to change the number of repetitions to get reasonable runtimes: many more for smaller messages, many fewer for larger messages. Each test needs to run for at least a few seconds to give reliable results.
   
 * The `halobench` program contains an example of using
   `MPI_Neighbor_alltoall()` to do pairwise swaps of data between neighbouring processes in a regular 3D grd
<!--- * Tomorrows traffic modelling problem sheet also contains a final MPI exercise
  in Section 3 to replace point-to-point boundary swapping with neighbourhood collectives. --->
 
* Collective exercises and solutions are in
  https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/exercises/collective.tar
  (instructions are included in comments at the top of each file).


<h4>Day 2</h4>

The reserved queue for today is called `ta206_1555702`. To use this queue, change the `--qos` and `--reservation` lines to:
````
#SBATCH --qos=reservation
#SBATCH --reservation=ta206_1555702
````

 * <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/exercises/traffic-advmpi.pdf">Traffic modeling exercise sheet</a>
 * <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/exercises/traffic.tar">Traffic model source code and solutions (MPI / OpenMP)</a>
  * <a href="https://github.com/EPCCed/archer2-AMPP-2025-07-29/raw/main/exercises/traffic-RMA.tar">Traffic model source code and solutions (MPI RMA)</a>

---

This work is licensed under a
[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License][cc-by-nc-sa].

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg

[![CC BY-NC-SA 4.0][cc-by-nc-sa-image]][cc-by-nc-sa]

