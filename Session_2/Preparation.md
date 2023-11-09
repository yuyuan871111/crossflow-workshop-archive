## Preparing your Archer2 environment for Crossflow

Here's a step-by-step guide to setting up your Archer2 environment so you can run Crossflow jobs on Archer2 from a web browser running on your laptop/desktop.

### Step 1: on your laptop/desktop:

Open a terminal window, and use it to log in to Archer2 in the usual way:

```bash
ssh <username>@login.archer2.ac.uk
```

### Step 2: Create and activate a fresh virtual environment on Archer2

The instructions here are adapted from those in the [Archer2 User Guide](https://docs.archer2.ac.uk/user-guide/).

It's important you create the virtual environment in your personal `/work` folder, not in your home directory. This is `/work/<projectname>/<projectname>/<username>` - here we assume `<projectname>` is *e280* :

```bash
module load cray-python
python -m venv /work/e280/e280/<username>/myvenv
source /work/e280/e280/<username>/myvenv/bin/activate
```

### Step 3 (on Archer2): Install the required Python packages

It's advisable to split this up into sections - trying to do it with one big `pip install ...` is risky. 

First update `pip` itself to avoid annoying warning messages:

```bash
pip install --upgrade pip
```

Next install `Jupyterlab`:
```bash
pip install jupyterlab
```

Next install the key workflow components - `crossflow` and `dask-jobqueue`:
```bash
pip install crossflow dask-jobqueue
```

Next install `mdtraj` - this will be used for manipulating MD (Gromacs) files:
```bash
pip install mdtraj
```

Finally install `mdplus` - this will be used for some structure-related calculations:
```bash
pip install mdplus
```

### Step 4 (on Archer 2): Configure your login environment for the workshop

Making sure you are in your home directory on Archer2, edit (or create if it doesn't exist yet) your `.bashrc` file to add commands to the end of it that will ensure when you next log in, all the software packages you need for this workshop will be available:

```bash
module use /work/e280/shared/hecbiosim-software/modules
module load cray-python
module load gromacs
module load autodock-vina-1.2.5
module load openbabel-3.1.1
source /work/e280/e280/crossflow001/myvenv/bin/activate
```

Log out of Archer2 and then back in again. You should find you are back in your virtual environment, and each of the following commands should give some output:

```bash
which gmx
which vina
which obabel
```

### Step 5 (on Archer 2): Start a Jupyterlab server

Move to your work directory:
```bash
cd /work/e280/e280/<username>
```
And use a text editor to create a shell script `jstart.sh` containing:

```bash
#! /usr/bin/env bash
# start a jupyterLab server
export JUPYTER_RUNTIME_DIR=${PWD}
jupyter lab --ip=0.0.0.0 --no-browser
```
Make it executable, and then run it:
```bash
chmod +x jstart.sh
./jstart.sh
```

Once the server has started, you should see some lines in the output that look something like:

```
...
Or copy and paste one of these URLs:
   http://<host>:<port>/lab?token=bdffd21a331e98f6e8c8da7941292d55921510edf10d4f9c
   http://127.0.0.1:<port>/lab?token=bdffd21a331e98f6e8c8da7941292d55921510edf10d4f9c
...
```
where `<host>` is something like *ln01* or *ln02* and `<port>` is something like *8888* or *8889*. You will need these for the next step.

You won't get a prompt back though - this terminal window is now busy, so you can move the window for it out of the way on your laptop screen (but don't close it - you will need it in a minute).

### Step 6 (on your laptop/desktop): Set up port forwarding

Open a second terminal session on your laptop, and connect to Archer2 again, but as follows:

```bash
ssh -L<port>:<host>:<port> <username>@login.archer2.ac.uk
```

You will be prompted for your password in the usual way. You won't use this terminal window any more, so again you can move the window for it out of the way on your laptop screen (but don't close it).

### Step 7 (on your laptop/desktop): Connect to your jupyterlab server

Open your browser, and paste the second of the two URLs printed when your Jupyterlab server started up (the one that begins `http://127.0.0.1`) into the address bar, and hit return. After a short while you should find yourself connected to your server, and positioned in your `work` directory on Archer.

From now on, everything that happens in this workshop will be done via this Jupyterlab connection. Once you have finished you will log out of this window, and then exit both terminal sessions to clean everything up. If you need to reatart things, just go back to Step 5 above.



```python

```
