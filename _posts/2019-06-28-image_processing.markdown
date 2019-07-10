---
layout: post
title:  "Parallel Image Processing using the Accelerator Part 1:  The Basics"
date:   2019-06-28 00:00:00
categories: processing
---

The Accelerator is designed for fast and reproducible data processing.
Typical application data is composed of text strings and numbers, but
the Accelerator can work as efficiently with any kind of binary data,
such as images or sound files.  In this post we will have a look at
batch processing of large quantities of images.  The focus will be on

  - parallel processing, i.e. how to make most use of the computer
    hardware to save execution time, and

  - reproducibility, i.e. how to associate results (output) to input
    data and source code.

We will use a simple example to show *how* to make best use of the
Accelerator's capabilities.  More advanced examples, such as neural
network inference, will be discussed in an upcoming post.





### Parallel Image Processing Example

A simple "easy to understand" example is that of creating image
thumbnails (i.e. downscaled copies) of a large set of images, see
figure.

<p align="center"><img src="{{ site.url }}/assets/image_rescale.svg"> </p>

Throughout this post, we'll assume that the number of parallel
processes is three (3) to keep things simple.  In an actual setup,
this number is equal or close to the number of available CPU cores on
the computer.

We will approach the example from two directions,

1. by keeping things simple and write a minimal parallel program to solve the task, and
2. by using more Accelerator features to create a solution that is
   more flexible and extendable.

The second approach will scale much better with an increased
complexity of problem to solve.  But let us start with the simple
straight-forward solution.





### Straightforward Parallel Program

The downscaling program receives a list of images to process as input,
and outputs a set of thumbnail image files.  To make most use of the
available hardware, the program will run in parallel on several CPU
cores.  Each parallel process will work on a unique slice of the input
image file list.  For each filename in the list, the process will read the
corresponding input image, downscale it, and write the output
thumbnail image, see the figure below.

<p align="center"><img src="{{ site.url }}/assets/image_files.svg"> </p>

Here is the complete source code for the program

```python
from PIL import Image

options=dict(files=[], size=(100, 100))

def analysis(sliceno, params):
    files = options.files[sliceno::params.slices]    # work on a slice of all filenames
    for fn in files:
        im = Image.open(fn)
        im = Image.thumbnail(options.size)
        im.save(fn + '.thumbnail', "JPEG")
```

The function `analysis()` will be forked and executed in
`params.slices` parallel processes.  Each process receives a unique
number between zero and the number of slices in the `sliceno`
variable.  Input options are the list of image files and the shape of
the output thumbnail image.

In order to execute the program, we need to write a small build script
containing the build rules

```python
def main(u):
    u.build('thumbnailer', options=dict(files=['file0.jpg', ...], size=(256,256)))
```





### Using More of the Accelerator's Features

The program in the previous section provides a simple but efficient
solution to the thumbnails task.  In this section, we'll introduce
Accelerator features that helps structuring a program and work on a
higher abstraction level with fast execution times.  Mainly, we'll

  - make use of the build system so that pre-computed results are
    reused and a minimum of processing is required when things (source
    code, parameters, or input data) change;

  - keep track of and connect intermediate results between programs
    automatically;

  - associate any number of additional pieces of information to each
    image; and

  - reduce the number of intermediate files, and thereby seek time

The first thing we do is to separate the solution into three different
programs.  We do this to make use of the Accelerator's _dataset_
storage format and to minimise the amount of re-builds when we modify
the code or input parameters.  Here is a build script reflecting the
partitioning of the code

```python
def main(u):
    files = sorted(glob.glob(os.path.join(path, 'file*.jpg')))
    jid_imp = u.build('import_images', options=dict(files=files))
    jid_tmb = u.build('thumbnailer',   options=dict(size=(256,256), datasets=dict(source=jid_imp)))
    jid_exp = u.build('export_images', options=dict(column='thumb', datasets=dict(source=jid_tmb)))
```

The most interesting program is the one in the middle, `thumbnailer`,
that actually computes the thumbnails.  The programs before and after
are just converting to and from the Accelerator's internal format.
The more complicated a processing task, the more this partitioning
makes sense, but we keep to the thumbnails example in this post to
keep things simple.

What about the `sorted()` call?  This is to ensure that we do not
execute any of the programs unless the input data has been modified.
Sorting the input data makes it deterministic, independent of how
`glob` or the file system works.  The `import_images`, and all jobs
depending on its output, will only execute once for a given input.  It
is only when inputs, parameters or source code change that programs
will be executed.  This is a key Accelerator feature.

Before we have a closer look at the `thumbnailer`, let's have a quick
look at the import program



#### The Import Program

The import program is much like the first thumbnailing program
presented earlier, but instead of writing output images to files, it
writes to an Accelerator _dataset_.  The dataset is used to store both
the images and some meta information.  In this case the meta
information is image filenames and the sizes.  If we had been
interested in, say, exposure statistics, we could add some or all of
the EXIF-data in addition to filename and size to the dataset.  The
import program is shown in the figure below

<p align="center"><img src="{{ site.url }}/assets/image_tods.svg"> </p>

Each process handles a slice of the list of input files and stores
them in a corresponding dataset slice.  Columns are stored in
independent files so that we can access columns independent of each
other.  Now we move on to the more interesting part.



#### The Image Processing part: Thumbnailing

With images and metadata available in a dataset, we can work on a
higher, yet efficient, abstraction level and focus on the actual image
processing flow.  For example, we can send the images to a set of
different image analysis algorithms, generate debug output image sets
for each of them, and merge all computations into a single result.
But in this post we focus on the thumbnail task.

The figure below illustrates how the `thumbnailer` program works

<p align="center"><img src="{{ site.url }}/assets/image_ds2ds.svg"> </p>

Each one of the parallel processes reads one slice of the `image`
column, creates a thumbnail, and then writes to a new colum named
`thumb`.  Note that

- the new column is appended to the existing "source" dataset, and
- there is no need to read any of the `shape` or `filename` data from disk.

After execution, the dataset has four columns, three that was created
by the import program, and one created by the thumbnail program.
Appending columns is almost for free, it just creates a link between
datasets.  By appending columns, everything we have computed and know
about each image is grouped together.  Not reading irrelevant columns
is obvious for performance reasons.



##### Diversion: Computing a Histogram of Image Shapes

It is tempting to show how easy it is to start doing data analysis now
that we have the imported image dataset.  The code below will compute
a histogram of all image shapes

```python
from collections import Counter
datasets = ('source',)
def synthesis():
    return Counter(datasets.source.iterate(None, 'shape'))
```

and we run it by adding this line to the build script

```
    ...
    u.build('shapehist', datasets=dict(source=jid_imp))
```

Here, `jid_imp` is a reference to the import job that was run
previously.  The `shapehist` program will thus always run on the
correct data.  Furthermore, `shapehist` makes use of the available
`shape` column in the dataset (which was generated as a by-product in
the import program), it does not need to read the images all over
again to compute the shapes, which has an enormous performance
advantage.




#### Exporting Images in a Dataset back to Files

Finally, we need a way to generate image files from a dataset.  Such a
program may be visualised like this

<p align="center"><img src="{{ site.url }}/assets/image_fromds.svg"> </p>

The program reads one line at a time from the `image` and `filename`
columns, and writes the image data into files with corresponding
filename.





### Source Code

Here is the source code for the programs above.  Note that the
Accelerator comes with a **minimum of dependencies**, and this is
considered to be a feature.  The Accelerator dataset supports a
variety of different types, but images is not one of them.  An
efficient way to handle images is to type them to the `bytes` type,
which supports arbitrary binary data up to 2GB per entry.
Unfortunately, there does not seem to be a Pillow method to generate
binary data from an image, so we use this wrapper

```python
from io import BytesIO
def pil2bytes(im):
    with BytesIO() as output:
        im.save(output, format="BMP")
        contents = output.getvalue()
    return contents
```

The choice of image codec is a trade-off between speed and storage.
BMP is a lossless format with fast encoding and decoding.



#### Import


#### Thumbnailer

#### Export


The Accelerator's internal dataset type


The main program will look something like this

It will read a column `image` and append a column `thumb` 

```python
from io import BytesIO

from . import pil2bytes

datasets=('parent',)

options=dict(size=(100, 100))

def prepare():
    dw = DatasetWriter(parent=datasets.parent)
    dw.add('thumb', 'bytes')
    return prepare_res

def analysis(prepare_res, sliceno):
    dw = prepare_res

    for im in datasets.source.iterate(sliceno, 'image'):
        im = Image.open(BytesIO(im)
        im.thumbnail(options.size)
        dw.write(helpers.pil2bytes(im))
```
The `pil2bytes` is necessary since PIL objects lack a way to create a bytestring directly.

```python 
from io import BytesIO
def pil2bytes(im):
    with BytesIO() as output:
        im.save(output, format="BMP")
        contents = output.getvalue()
    return contents
```


`depend_extra`

### Additional Resources

[The Accelerator's Homepage (exax.org)](https://exax.org)  
[The Accelerator on Github/eBay](https://github.com/ebay/accelerator)  
[Installation Manual with Performance Test](https://berkeman.github.io/pdf/acc_install.pdf)  
[Reference Manual](https://berkeman.github.io/pdf/acc_manual.pdf)  
