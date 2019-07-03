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



### Parallel Image Processing
There are two fundamentally different approaches to storing images for
parallel processing using the Accelerator:

  1. as individual files, or
  2. using the Accelerator's datasets for streaming processing

The second approach, using datasets, is the preferred choice, but
we'll have a look at both to see how they work.



#### Using Independent Image Files
Let's assume that we want to create thumbnails of a large number of
jpeg images stored in a certain directory.  To speed up the
processing, we want to processes images in parallel.  Using the
Accelerator it is trivial to parallelise processing.

To keep numbers low, let's assume that we are going to use three
parallel processes.

<p align="center"><img src="{{ site.url }}/assets/image_files.svg"> </p>


The job takes the path to the images and the thumbnail size as input options

```python
from glob import glob
from os.path import join
from PIL import Image

options=dict(path='', size=(100, 100))

def prepare():
    return glob(join(options.path, '*.jpg'))        # return list of all image filenames

def analysis(prepare_res, sliceno, params):
    files = prepare_res[sliceno::params.slices]     # work on a slice of all filenames

    for fn in files:
        im = Image.open(fn)
        im = Image.thumbnail(options.size)
        im.save(fn + '.thumbnail', "JPEG")
```
That is all.  This job will process the images in parallel!

#### Using the Accelerator's Dataset Storage Format

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
