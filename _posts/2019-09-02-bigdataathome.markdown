---
layout: post
title:  "Bigdata at Home"
date:   2019-09-02 00:00:00
categories: performance
---

How to approach the problem of processing large datasets?  Does it
require expensive hardware?  Is it slow?

In this post we'll show how to do fast and reliable parallel computing
of large datasets for less than $1000.

Our approach is Accelerator based.  The Accelerator uses available
hardware resources efficiently, and many applications will therefore
run well on a standard laptop.  But a more powerful computer will
reduce execution time and shorten the design iteration time.



### The Accelerator Framework

The Accelerator is a minimalistic framework with a small footprint
that runs on anything from 32-bit Raspberry Pies to 64 bit multi core
rack servers.  One of many features is that it provides simple
parallel processing that makes good use of modern parallel computer
hardware.  Since the Accelerator is programmed in Python, it is easy
to use and get familiar with.



### Hardware for the Accelerator

As mentioned, the Accelerator runs on almost any platform.  For many
data processing and analysis applications, performance scale well with
the number of CPU cores, so more cores is better.  Plenty of memory is
also beneficial for disk buffering of intermediate and previous
results.  Large disks are needed if the datasets are large.

We believe that memories with parity control (ECC) and file systems
with integrity checking and reduncancy (such as zfs) is a requirement
to fulfill the reproducability requirement.  Any error in the data
should at least be detected (and preferably corrected).



### Our Pick:  The Second-Hand Lenovo D20 Workstation from 2011

It is cheap, reliable, and powerful. It actually compares well to
modern machines, which we will show in _another post_.

The 2011 Lenovo D20 workstation comes with two CPUs providing in total
12 cores (24 "threads").  It has 108GB RAM and 2TB solid state disk.
We found this item on eBay, approx price was **below $1000**.

<p align="center"><img src="{{ site.url }}/assets/ThinkStation D20 014.JPG" height="300"> </p>

If used at full capacity, a machine like this, from 2011, beats the
most powerful laptop from 2018.  This is mainly because it does not
need to do aggresive thermal CPU frequency throttling, and it can
address memory faster since it has three independent memory channels
per CPU chip.

| cpu | 12 cores | X5675 @ xx GHz |
| ram | ddr3 | 108GB |
| disk | ssd| 2TB|

The machine clearly has too little disk for data intensive
applications, but it is straightforward to have in total eight drives
in the box.  We have measured the disk controllers to transfer data
at a sustained rate of 265MB/s to and from the SSDs.



### Performance Testing

The Accelerator installation repository comes with a simple
performance testing script that we've used for the testing.  The
details can be found on page 3 of [this
document](https://berkeman.github.io/pdf/acc_install.pdf).  Basically,
what happens is that the test will create a dataset of one billion
lines, and this dataset will be read, written, and processed in
different ways while execution time is measured.


The size of the dataset is 79 GiB (36 GiB compressed).


### Test Results

Here is the complete output from the `example_perf` build script.

```
      operation                       exec time         rows/s

1.    csvexport                         686.775      1,456,081

2.    reimport total                    912.549      1,095,832
         csvimport                      591.277      1,691,256
         type                           321.272      3,112,623

3.    sum
        small number                      4.192    238,556,448
        small integer                     3.156    316,887,361
        large number                     10.907     91,688,275
        gauss number                      5.519    181,178,210
        gauss float                       4.345    230,175,033

4.    sum positive
        small number                     11.814     84,643,880
        small integer                    10.683     93,603,000
        large number                     18.803     53,181,776
        gauss number                     14.636     68,322,432
        gauss float                      13.615     73,450,887

5.    histogram
        number                           45.894     21,789,231
        float                            44.249     22,599,181

6.    find string                        12.304     81,275,676

7.    Total test time                  3235.222

Example size is 1,000,000,000 lines.
Number of slices is 23.
```

In less than one hour (3235 seconds), it reads and writes as well as
operates on a billion lines of data, several times.  Let us take a
closer look at what has happened:

#### 1. csvexport

The first thing that happens in the script is that it creates 100
chained datasets with 10 million rows each.  All this data is fed to
the `csvexport` method that writes it to a CVS file on disk.
Exporting the data takes about 11 minutes (687 seconds) at a rate of
almost 1.5 million rows per second or 115 MB/s

#### 2. reimport

The CSV file is then imported again using the `csvimport` method,
followed by typing of the data using `dataset_type`.  Importing runs
at 1.7 million rows per second while typing runs at 3.1 million rows
per second.  (The typing jobs both reads and writes about 250MB per
second in parallel!)

#### 3. sum

Now the imported data is used for calculations.  The script begins
with adding single columns together.  The test shows the difference in
speed between different datatypes.  For example, the Acclerator adds
230 million gaussian floats per second.  The "small integer" dataset
has less entropy and runs at 317 million rows per second.

There are two main reasons for this high performance.  First, the
Accelerator only reads the columns needed for the current work, and
not all the data.  Second, data is compressed on disk, which makes it
possible to "go beyond the IO bandwidth" between disk and memory.

#### 4. sum positive

This test adds a data dependency.  Numbers are added only if they are
positive.

#### 5. histogram

This part uses the Python `Counter` class to store distributions of
each type.  Although it uses a very high-level language construct, it
runs at more than 20 million rows per second.  We can create a
histogram of a billion floats in about 45 seconds!

#### 6. find string

Here we check for existence of a four character string in a billion
random strings of length X.  We can do this at a rate exceeding 80
million rows per second.  (The number of strings found is X out of a
billion, which is very close to the theoretical result which is Y.)

#### 7. Total Test Time

... and we are done in less than one hour.



### Conclusion

The Accelerator can handle large datasets at high speed on inexpensive
hardware, and this is one of the reasons it is the ideal framework for
a lot of data intensive work.
