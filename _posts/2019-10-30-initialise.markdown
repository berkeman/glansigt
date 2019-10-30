---
layout: post
title:  "Accelerator Installation"
date:   2019-10-30 00:00:00
categories: example
---


This HOWTO shows how to install and test the Accelerator.

#### TL;DR
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

Project source files will go into the `dev/` directory, and we'll talk
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
how many processes the Accelerator will run in parallel.  The
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
mandatory, but it is good practice to use it to avoid surprises.

Then follows the definition of which [packages](https://docs.python.org/3/tutorial/modules.html#packages)
that should be accessible in the project:

```text
method packages:
        dev
        accelerator.standard_methods
        accelerator.test_methods
```

Three packages are defined here, first the `dev` package (which
corresponds to our recently created `myproject/dev` directory), and
then two packages from the Accelerator installation:
`standard_methods` and `test_methods`.

Finally, this section:
```text
result directory: ${HOME}/accelerator/results
source directory: # /some/path where you want import methods to look.
logfile: ${HOME}/accelerator/daemon.log
```

specifies two directories and a log file.  Unless these are used
explicitly, it is only the log file that will matter here.



#### The dev package

Let's go back to the `dev/` directory.  This is where the project's
source files are to be stored.  The directory is actually a Python
package (that can be `import`ed), and thus it contains an `__init__.py`
file.  Here is a complete list of the files:

```text
dev/
    __init__.py
    a_example.py
    automata.py
    methods.conf
```

The directory contains one example method `a_example.py` and one
example build script `automata.py`.  It also contains the mandatory
`methods.conf` file which specifies which of all methods in the
package that should be executable.  The file contains the single line

```text
example
```

indicating that `a_example.py` is indeed executable in the current
project.



### Testing the Installation

If we are happy with the workdir definition of the configuration file
we can proceed and run the built in tests.  If not, this is the time
to edit `accelerator.conf`.

The Accelerator is based on a "client-server" architecture.  
Make sure the virtual environment is active, and do

```shell
ax daemon
```

This starts the server.  It is highly recommended to keep the server
running in a separate terminal, so when issuing commands to it, we do
that from a new terminal.

So, create a new terminal (gnu screen or tmux is suggested), initiate
the virtual environment and `cd` to the project's directory:
```shell
source accvenv/bin/activate
cd myproject
```

Now we can run the built in tests, like this:

```shell
ax run tests
```

This will run `automata.tests` which the Accelerator searches for (in
alphabetical order) in all packages specified in the configuration
file.  In this case it is located in the `example_tests` package.
**This might fail**, please read on!

The `tests` script will try a number of corner cases relating to for
example character encoding, and it will break if there is insufficient
support of _locales_ installed on the system.  In particular, if you
get

```text
Exception: Failed to enable numeric_comma, please install at least one
of the following locales: da_DK nb_NO nn_NO sv_SE fi_FI en_ZA es_ES
es_MX fr_FR ru_RU de_DE nl_NL it_IT
```

this is perfectly fine.  On a Debian-based system, locales can be
configured using

```sh
dpkg-reconfigure locales
```

-Why?  Well, the Accelerator is designed to handle in practice any
character encoding scheme, and in order to guarantee it, it has to be
tested.  If the test fails, well, this indicates that the system might
not support all cases that the Accelerator is designed to handle.


### Install from the git Repository

It is also possible to install directly from the git repository.

Clone the repository
```sh
git clone https://github.com/eBay/accelerator
```
and install
```sh
cd accelerator
./setup.py build
./setup.py install
```

On a Debian-based system, the dependencies are @@@@@@@@@@@@@@@@@@@@@@@@@
