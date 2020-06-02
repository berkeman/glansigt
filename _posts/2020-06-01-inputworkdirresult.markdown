---
layout: post
title:  "Save Time, Keep Order, Be Traceable:<br>  From Input Data to Result"
date:   2020-06-01 00:00:00
categories: example
---


One key feature with the Accelerator from eBay is its *build system*
and the way it handles intermediate storage of data and parameters.
The goal of the system is to provide **transparency** and
**reproducilility** by design, to any project.  While being main
features on their own, this also reduces the risk of making errors,
while typically speeding up execution times, both in the development and deployment phases.

Processing is "from left to right", starting with input data end
ending with output results.  Inbetween, any intermediate or temporary
storage is written (and read back) to a dedicated space called the
*workdir*, see figure below.

<p align="center"><img src="{{ site.url }}/assets/input_results_splash.svg"> </p>

This post will discuss what problems the Accelertor's build system
targets, and how it solves them.



# Result = Input + Program + Execution time

Typically, a data science task is about creating some output from a
data set, like this

<p align="center"><img src="{{ site.url }}/assets/input_output.svg"> </p>

The output could be some insight in form of a graph or number, some
kind of model of the data, a subset of the data, or something else, it
does not matter.  The point is that the result is a function of the
input data.

Conceptually, the output is created by a program acting on the input,
something like this

<p align="center"><img src="{{ site.url }}/assets/input_prog_output.svg"> </p>

From this image, it is also obvious that the result does not only
depend on the input data, but also on the actual program used to
generate the result.

We also note that it takes some _execution time_ to generate the
result from the input data using the program.



# Speeding up by Partitioning the Program

As long as the input dataset is small and execution time is short, the
single program approach works very well.  But it will be increasingly
more difficult to develop the program when the dataset is larger and
it takes a significant amount of time to run a single iteration.

To overcome the issue of longer execution times, it is common to apply
an iterative approach.  We can probably split the program into
different parts, where input to one part is the output of another,
like this.

<p align="center"><img src="{{ site.url }}/assets/input_prog_output2.svg"> </p>

The main benefit here is that once we've developed and debugged
program1, we can store its output on disk and concentrate on the
development of program2 _without_ running program1 again.

Assuming that writing and reading the intermediate storage is fast,
this may save a considerable amount of execution time.



# The Problem of Versions

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

<p align="center"><img src="{{ site.url }}/assets/input_results.svg"> </p>


# The Question

How to **convert 68.000 JSON files** into a single CSV file in a
reproducible and traceable way **in one minute** on a laptop.


# Introduction

In the popular "COVID-19 Open Research Dataset Challenge (CORD-19)" on
[Kaggle](https://www.kaggle.com/allen-institute-for-ai/CORD-19-research-challenge),
the challenge is to extract information from a dataset composed of a
large number of scientific papers.  While the Natural Language
Processing (NLP) part of this project is the main focus of the Kaggle
challenge, this post covers the important pre-processing of the
dataset.

The complete source code is available on
[github](https://github.com/exaxorg/Kaggle-CORD19-data-parser).



# Pre-Processing

The Kaggle dataset is composed of JSON-files, one file per paper.  In
total there are more than 68.000 files spread over several
directories.  The pre-processing proposed her converts all these files
into one Comma Separated Values (CSV) file.  (Actually, the separator
could be any character.)  Various NLP-preprocessing tricks such as
conversion to lowercase and tokenisation can also be carried out in
the process.

Why does this matter?  Mainly because

 - reading a single CSV file is faster than reading 68.000 JSON-files
   from different directories.

 - the resulting CSV file, and thus all further processing, becomes
   independent of the input data format, that may change.

 - tokenisation, conversion to lowercase etc. is carried out once and
   separated from all further processing, which is faster.

 - a CSV-file can be manipulated by standard shell tools such as
   `grep` for visualisation and validation tasks.

A single file is also easier to share.

In addition, we want the pre-processing to be fully reproducible and
transparent.  It should be possible to run on several different
versions of the dataset and algorithms without difficulty.



# The Code

The source code is available
[here](https://github.com/exaxorg/Kaggle-CORD19-data-parser).  It is using the
Accelerator data processor from eBay for parallel and reproducible
computing.

It is all very simple.  There is only one custom method,
`dev/a_import.py`, which reads all JSON files in parallel, parses
them, and writes them to a common dataset.  The method is called once
for every directory containing JSON files, and all resulting datasets
are chained.  Finally, the bundled `csvexport` method is used to
convert the resulting dataset chain into a CSV-like file.  This method
has options for selection of separator, citations, and more.



# Example Execution time

On a 2018 high-performance laptop (HP Elitebook 840 G5), the complete
processing takes **61 seconds**.

(A [2009 Lenovo
workstation](https://expertmakeraccelerator.org/performance/2019/09/02/bigdata_on_inexpensive_workstation.html)
is more than twice as fast and at half the price of the laptop!)



# About the Accelerator

The Accelerator from eBay is designed for fast processing of large
datasets on a single machine.  Programs are simply written in Python,
and thanks to its fast parallel processing and clever reuse of
pre-computed results, execution times are typically in the range of a
few seconds per task.  Therefore, ideas could be tried out with very
little overhead, making the Accelerator a perfect tool for fast
exploration of large datasets.

