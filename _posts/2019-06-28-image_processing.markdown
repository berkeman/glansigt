---
layout: post
title:  "Parallel Image Processing using the Accelerator"
date:   2019-06-28 00:00:00
categories: processing
---

The Accelerator is ideal for batch processing of large quantities of
images for two reasons

  - processing is carried out in parallel, and
  - book keeping of relations between input and output images is automatic.

in this post we will look at how this can be carried out in practice.



### Parallel Image Processing Example

Let's assume that we want to create thumbnails of a large number of
jpeg images stored in a certain directory.  This is an operation that
is ideal for parallel processing, since resampling an image is a
computationally expensive task and images are independent of each
other.

At setup time, the Accelerator is configured with the number of
parallel processes to use.  Typically, this number is equal or close
to the number of available CPU cores on the computer.  To keep things
simple, let's assume that this number is three.



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

Here is the complete source code

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

Input options are the list of image files and the shape of the output
thumbnail image.  The function `analysis()` will be forked and
executed in `params.slices` parallel processes.  Each process receives
a unique number between zero and the number of slices in the `sliceno`
variable.  This program is all that is needed.




#### Using the Accelerator's Dataset Storage Format

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

<p align="center"><img src="{{ site.url }}/assets/image_ds2ds.svg"> </p>



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
