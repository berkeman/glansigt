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

The previous example solves the thumbnails task, and can easily be
expanded with more features.  For more advanced tasks, however, we can
make use of a larger set of the Accelerator's capabilities, such as

  - making full use of the build system so that pre-computed results
    are reused and a minimum of processing is required when things
    change.

  - keep track of intermediate results

  - associate any number of additional pieces of information to each
    image

  - reduce the number of intermediate files, and thereby seek time

Here is a build script showing how we partition the problem for more
flexibility

```python
def main(u):
    files = sorted(glob.glob(os.path.join(path, 'file*.jpg')))
    jid_imp = u.build('import_images', options=dict(files=files))
    jid_tmb = u.build('thumbnailer',   options=dict(size=(256,256), datasets=dict(source=jid_imp)))
    jid_exp = u.build('export_images', options=dict(column='thumb', datasets=dict(source=jid_tmb)))
```

The most interesting program is the one in the middle, `thumbnailer`,
that actually computes the thumbnails.  The programs before and after
are just converting to and from the Accelerator's internal formats.
The more complicated a processing task, the more this partitioning
makes sense, but we keep to the thumbnails example in this post to
keep things simple.

What about the `sorted()` call?  This is to ensure that we do not
execute any programs unless the input data has been modified.  Sorting
the input data makes it deterministic, independent of how `glob` or
the file system work.  The `import_images`, and all jobs depending on
its results, will only execute once for a given input.  This is a key
Accelerator feature.

Before we have a closer look at the `thumbnailer`, let's have a quick
look at the import program



#### The Import Program

The import program is much like the previous example, but instead of
writing output images to files, it writes to an Accelerator dataset.
It stores the images as well as some meta information, in this case
the filenames and the image sizes.  If we had been interested in, say,
exposure statistics, we would add some or all of the EXIF-data in
addition to filename and size to the dataset.  The import program is
shown in the figure below

<p align="center"><img src="{{ site.url }}/assets/image_tods.svg"> </p>

Each process handles a slice of the list of input files and stores
them in a corresponding dataset slice.  Columns are stored in
independent files.  Now we move on to the more interesting part.



#### The Image Processing part: Thumbnailing

Now that we have images and meta data in a dataset, we can work on a
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

Appending columns is almost for free, it just creates a link between
datasets.  The advantage we get from appending columns is that we keep
everything we have computed and know about each image together.  Not
reading irrelevant columns is obvious for performance reasons.


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
correct data.




#### Exporting Images in a Dataset back to Files

Finally, we need a way to generate image files from a dataset.  Such a
program may be visualised like this

<p align="center"><img src="{{ site.url }}/assets/image_fromds.svg"> </p>

The program reads one line at a time from the `image` and
`filename`-columns, and write the image data into files with
corresponding filename.





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
