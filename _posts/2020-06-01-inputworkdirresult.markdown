---
layout: post
title:  "Save Time, Keep Order, Be Traceable:<br>  From Input Data to Result"
date:   2020-06-01 00:00:00
categories: example
---


One key feature of the Accelerator from eBay is its *build system* and
the way it stores and keeps track of all programs executed and their
generated output.  The goal of the system is to provide
**transparency** and **reproducilility** by design, to any project.
While these are main features on their own, the design also reduces
the risk of making errors, while typically speeding up execution
times, both in the development and deployment phases.

The improvement in speed is because the Accelerator makes sure that
the _same program can not be executed twice on the same data_.  If a
result has been computed before, it will be re-used instead of
re-computed.  (Another, unrelated, reason for the Accelerator to being
fast is that it provides a parallel streaming data interface that it is
embarrasingly simple to write parallel programs around, in plain
Python.)

The figure below is a high level illustration of what the
Accelerator's processing flow looks like

<p align="center"><img src="{{ site.url }}/assets/input_results_splash.svg"> </p>

Data processing is "from left to right", starting with input data end
ending with output results.  Inbetween, any intermediate or temporary
storage is written (and read back) to a dedicated space called the
*work directory*, see figure below.

This post explains how the Accelerator's build system works, what kind
of problem it targets, and how it solves them.



# Result = Input + Program + Execution time

Let us start with the basics to make sure we are on the same page.
Typically, a data science task is about creating some output from a
data set, like this

<p align="center"><img src="{{ site.url }}/assets/input_output.svg"> </p>

The output could be some insight in form of a graph or number, some
kind of model of the data, a subset of the data, or something else.
It does not matter what it is, the point here is that the result is a
function of the input data.

Conceptually, the output is created by a program acting on the input,
something like this

<p align="center"><img src="{{ site.url }}/assets/input_prog_output.svg"> </p>

From this image, it is also obvious that the result does not only
depend on the input data, but also on the actual program used to
generate the result.  And, if the program takes input options, the
result may depend on these as well.

We also note that it takes some _execution time_ to generate the
result from the input data using the program.  If the program runs
fast and the dataset is small, this is perhaps not a big issue.  We
can keep track of the complete process from input to output using a
single _Makefile_ for example.  When the source code is updated we can
update the result correspondingly by running _make_.  But if the
program runs for a longer time, development cycles will be longer and
results will be more valuable since it takes time to generate them.



## Speeding up by Partitioning the Program

Development and debugging of a program becomes increasingly difficult
with larger datasets and longer execution times.  To overcome the
issue of longer execution times, it is common to apply an iterative
approach.  We can probably split the program into different parts,
where input to one part is the output of another, like this.

<p align="center"><img src="{{ site.url }}/assets/input_prog_output2.svg"> </p>

The main benefit here is that once we've developed and debugged
program1, we can store its output on disk and concentrate on the
development of program2 _without_ running program1 again.  Assuming
that writing and reading the intermediate storage is fast, this may
save a considerable amount of execution time.

Another benefit might be that the output data from program1 may be
used for several applications, so we can fork different subprojects
from the output of program1 instead of starting over at the input data
file.  If program1 does some data preprocessing or cleanup, for
example, this makes total sense.  In fact, we probaly want to _ensure_
that the same preprocessing is applied to all our subprojects.  "Don't
Repeat Yourself".




## The Problem of Versions

While program partitioning, as shown in the previous section, is a
common approach that seems simple enough, it is actually a road to
trouble.  Why is that?  Well, it usually does not take long before we
have something like in the figure below (or even more complex)

<p align="center"><img src="{{ site.url }}/assets/input_prog_output3.svg"> </p>

Here, we've made two changes.

 - First, program1 was modified.  This modification caused the output
to change as well.  The updated output had to be stored on disk so
that program2 could use the updated instead of the old version.
Running program2 on the updated result caused the output result to change.

- Second, program2 was modified.  This caused the result to change again.

In total, we now have three different results.  There are several
potential problems here, for example (tick the ones you recognize):

- We have to manually map the different results to the different
  versions of the program that we have run.  Potentially, the input
  data has been modified too, so we need to keep track of the data
  version as well.  Manually.

- Maybe we have overwritten old results with new ones "as we go", and
  in that case there is nothing left for us to compare our latest
  results to.  Maybe we did a cut and paste to a text file or spread
  sheet.  But which version of the code did it match again?

- Are we sure the result is based on the current version of the data
  and code?  If not - run everything again just to be sure (*sigh*)!

- What if we decide that the first version is the one we want to keep?
  If we've overwritten our intermediate data, we have to re-run the
  first version *again*.  Time consuming.  Could we have kept all
  outputs and results?  -Yes we could - disk space is very cheap these days.

It seems we have to _remember_ or _manually keep track of_ which
result that belongs to which version of the code.  **This does not
scale well**, not with complexity, and not over time, since we
probably forget what is what after a while.

Apparently the approach is not *transparent*, not necessarily
*reproducible*, and since it relies on human book keeping it is
definitely *easy to make mistakes*.



# The Accelerator Approach

The approach taken by the Accelerator is simple but very efficient.
Running a program using the Accelerator is considered to be an atomic
operation.  In Accelerator lingo this is called to run a _job_.
Everything related to a program run is kept together in one place for
reference and transparency.  This means that any intermediate files in
a project are tied to the job that created them.

In practice, when a new program is run, the Accelerator creates a
directory (called a _job directory) which is populated with files
covering all information that relates to the program run.  The first
things that are written in the directory are files contain meta
information about the job.  Thereafter, any files written by the
running program as well as the program's return data and other output
is stored in the directory.  In brief, the job directory contains
among other things

 - the program's source code,
 - all input parameters,
 - everything written to stdout and stderr,
 - all output files generated by the running program,
 - the returned value of the program
 - exectime and profiling information, and
 - a hash digest uniquely identifying the job.

In fact, a job directory contains everything needed to run a specific
program with a particular set of input parameters and data references.
It also contains anything output during the program's execution,
including temp files, print-outputs, stderr-messages and return value.

When the program completes execution, the Accelerator returns a link
(or a path) to the job directory back to the caller.

--- Jobs can only be build once, fetch and re-use old jobs, never execute the same thing twice ---


The figure below is a high level view of the Accelerators data flow

<p align="center"><img src="{{ site.url }}/assets/input_results.svg"> </p>

Data is read from the `input directory`, which is set in a
configuration file.  All jobs are stored in the `work directory`.  The
work directory is in fact a database of all executed jobs, implemented
using plain directories and files for transparency and efficiency.

Since everything produced by a program will be stored inside a job
directory, all resulting files and data, such as for example diagrams,
will be located inside job directories.  To make it straightforward to
find these, they can be soft linked to the `result directory`.

Thus, data is read from files in the `input directory`, intermediate
files are stored in the `work directory`, and results are placed in
the `result directory`.  Plain and simple.



## Transparency

By looking at a resulting file, one can directly see the link to the
file inside a job directory.  inside this job is the source code that
generated the file, along with references to input data and other jobs
that were used to process the data.  Following all links backwards
will unwind the complete processing graph, independent of how complex
it is, all the way back to the input data.  For each job, the source
code and parameters can be extracted too.



# About the Accelerator

The Accelerator from eBay is designed for fast processing of large
datasets on a single machine.  Programs are simply written in Python,
and thanks to its fast parallel processing and clever reuse of
pre-computed results, execution times are typically in the range of a
few seconds per task.  Therefore, ideas could be tried out with very
little overhead, making the Accelerator a perfect tool for fast
exploration of large datasets.

