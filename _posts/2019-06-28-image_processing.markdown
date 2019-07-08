---
layout: post
title:  "Parallel Image Processing using the Accelerator Part 1:  The Basics"
date:   2019-06-28 00:00:00
categories: processing
---

The Accelerator is ideal for batch processing of large quantities of
images for two main reasons:

  - Processing is carried out in parallel, and
  - book keeping of relations between input and output images is automatic.

How this is carried out in practice is the topic for this and the next
post, "NN something something..".  This post will provide the basics,
and the next will show...





### Parallel Image Processing Example

We'll start with something simple.  Let's assume that we want to
create thumbnails of a large number of jpeg images stored in a certain
directory.  This task is ideal for parallel processing, since
resampling an image is a computationally expensive task and images are
independent of each other.

Throughout this post, we'll assume that the number of parallel
processes is three (3) to keep things simple.  In an actual setup,
this number is equal or close to the number of available CPU cores on
the computer.
OA


### Two Storage Models

The Accelerator is designed for parallel processing.  It also provides
efficient data storage formats and operators for high speed stream
processing of large datasets.  In this post, we'll have a brief look
at both approaches, that is, we can either store images

  1. as individual image files, or
  2. using the Accelerator's datasets for streaming processing.

The second approach, using datasets, is the more flexible choice, but
for some small work tasks it might do as well using plain files.



#### Using Image Files Directly

Our parallel program receives a list of images to process as input.
Each process will slice this list into a unique subset of images to be
considered by that particular process.  Each process reads,
transforms, and writes one image at a time, see the figure below

<p align="center"><img src="{{ site.url }}/assets/image_files.svg"> </p>

And here is the source code

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

In order to execute the program, we need to write a small build script
containing the build rules

```python
def main(u):
    u.build('thumbnailer', options=dict(files=['file0.jpg', ...], size=(256,256)))
```

Input options are the list of image files and the shape of the output
thumbnail image.  The function `analysis()` will be forked and
executed in `params.slices` parallel processes.  Each process receives
a unique number between zero and the number of slices in the `sliceno`
variable.




#### Using More of the Accelerator's Features

The previous example is just what is needed to solve the thumbnails
task, and it is easy to expand it with more features.  For more
advanced tasks, such as ..., however, we can improve by

  - making use of the build system to make sure that output from previously built jobs is reused

  - keep track of intermediate results

  - associate any number of additional pieces of information to each image

  - reduce the number of intermediate files, and thereby seek time


Here is a build script showing how we partition the problem

```python
def main(u):
    files = glob.glob(os.path.join(path, 'file*.jpg'))
    jid_imp = u.build('import_images', options=dict(files=files))
    jid_tmb = u.build('thumbnailer',   options=dict(size=(256,256), datasets=dict(source=jid_imp)))
    jid_exp = u.build('export_images', options=dict(column='thumb', datasets=dict(source=jid_tmb)))
```

The most interesting program is the one in the middle, `thumbnailer`,
that actually computes the thumbnails.  The programs before and after
are just converting to and from the Accelerator's internal formats.




Now why on earth do we want to complicate things like this?


The figure below illustrates how the `thumbnailer` program works

<p align="center"><img src="{{ site.url }}/assets/image_ds2ds.svg"> </p>

Each one of the parallel processes reads one slice of the `image`
column, creates a thumbnail, and then writes to a new colum named
`thumb`.  Note that

- the new column is appended to the existing "source" dataset, and
- there is no need to read any of the `shape` or `filename` data from disk.

Appending columns is almost for free, it just creates a link between
datasets.  The advantage we get from appending columns is that we keep
everything we have computed and know about each image together.  Not
reading irrelevant columns is obvious for performance reasons.




Now, we'll turn to using the Accelerator's internal dataset type for
internal representation of images.  Datasets are efficient
abstractions that simplifies the design of more complicated programs,
so what we do next can be seen as a preparation for writing something
much more complex than thumbnail generation.  We are going to write
three programs, first two helpers that

  - read images from a directory and "imports" them to an Accelerator dataset, and
  - converts a dataset of images to a set of image files.

and then 

  - the actual processing, that reads and appends to a dataset.

The interesting things happen in the processing job, which we will get
to shortly, but first we briefly have a look at the two helper programs.



##### Importing Image Files into a Dataset

This program is much like the previous example, but instead of writing
output images to files, it writes to a dataset.  It stores the images
as well as some meta information, in this case the filenames and the
image sizes, see figure below

<p align="center"><img src="{{ site.url }}/assets/image_tods.svg"> </p>

Each process handle a slice of the list of input files and stores them
in a corresponding dataset slice.  Columns are stored in independent
files.




##### Exporting Images in a Dataset back to Files

The corresponding "write to image files" program may be visualised
like this.  The program reads the `image` and `filename`-columns...

<p align="center"><img src="{{ site.url }}/assets/image_fromds.svg"> </p>


Visually, the programs may be represented as follows





xx


xx




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
