---
layout: post
title:  "Accelerator Installation"
date:   2019-10-30 00:00:00
categories: example
---


This HOWTO shows how to install and test the Accelerator.

#### TD;DR
```sh
# install
pip install accelerator
# setup new project
mkdir myproject
cd myproject
ax init
# check/modify config file according to your needs
ax daemon &
ax run tests
```



### Install Using pip

Using pip, the Accelerator is installed just like any other package.
Installation can be global, local, or in a virtual environment.
Here we show how to set up in a virtual environment.

The first thing is to create and activate a virtual environment.  We
do this using Python's `venv` package like this

```shell
python3 -m venv accvenv
source accvenv/bin/activate
```

The next step is then to install the Accelerator

```shell
pip install accelerator
```

This command will download, compile, and install the Accelerator to
the active virtual environment.



### Set up a Simple Project

Now that the Accelerator is installed, let's create a simple project.
Each project should have its own directory, so we use `mkdir` to
create a new directory, and then we `cd` into it

```shell
mkdir myproject
cd myproject
```
and run the Accelerator's init command
```shell
ax init
```

This will set up everything needed to work on a new project.  **At
this point we are done**, but please read on to learn about how to
tweak the configuration file and how to add files to a project.



### What's in the Project Directory

If we do a `ls -F` in the `myproject` directory, we'll find one new
file and one new directory

```text
accelerator.conf  dev/
```

Project source files will go into the `dev` directory, and we'll talk
more about that later.

The file `accelerator.conf` contains the project's configuration.
This file is read every time an Accelerator command is executed, so
when working on the project, the `myproject` directory should be the
"current working directory".  Let's have a look at its content.



#### The Configuration File

_The configuration file comes with pretty detailed inline documentation
that we've chosen to not show here for brevity reasons.  Instead,
we'll walk through the file and comment the things that matters in
this project right now._

First, the number of _slices_ is specified.  This number stipulates
how many processes the Accelerator will run in parallell.  The
variable is initiated to the number of available CPU cores minus one:

```text
slices: 3
```

Then, all workdirs are defined:
```text
workdirs:
        dev ${HOME}/accelerator/workdirs/dev
		
target workdir: dev
```

A workdir is where jobs and their output is stored.  Any number of
workdirs can be specified here, and workdirs can be shared between
users.  In this case a singe workdir `dev` is defined that is located
in the directory `${HOME}/accelerator/workdirs/dev`.  (The `${HOME}`
is a reference to the shell environment variable `HOME`.  Absolute
paths are of course also fine.)  The `target workdir` is not
mandatory, but good practice to avoid surprises.

Then follows the definition of packages:
```text
method packages:
        dev
        accelerator.standard_methods
        accelerator.test_methods
```



result directory: ${HOME}/accelerator/results
source directory: # /some/path where you want import methods to look.
logfile: ${HOME}/accelerator/daemon.log
```
						


The `dev` directory is a _python package_ where project source files
should be located.

explain files/dirs

edit config, define workdirs etc, berätta om att tests finns


```shell
ax daemon
```


```shell
ax run tests
```


```text
dev/
    __init__.py
    a_example.py
    automata.py
    methods.conf
```

sluta med eget exempel
