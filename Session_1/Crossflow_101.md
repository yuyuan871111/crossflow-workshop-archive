# Crossflow 101

An introduction to building workflows with Crossflow.

### Prerequisites:

* Access to a Jupyter notebook

### Summary

The purpose of Crossflow is to allow you to write workflows in Python. There are three basic components:

* Crossflow *tasks* allow you to convert pretty much any command-line tool into a Python function.
* Crossflow *clients* allow you to execute these tasks on (maybe) remote computing resources.
* Crossflow *filehandles* provide a way to pass data between the tasks, without relying on the filesystem.

Here you will see all three components in action. Usually workflows stitch together complex and compute-intensive steps, but we can illustrate all the principles using a very simple example:

* You will create *tasks* for three simple unix commands - *split*, *tac* (or *tail*), and *cat*
* You will create a *client* to run the tasks on a local *cluster* (actually, the same machine you are working on).
* You will see how to use *filehandles* to upload, download, and inspect data.

Your workflow will do the following:

1. Take a text file, and reverse it line-wise.
2. Split the reversed file into five chunks.
3. Reverse each of the five chunks (so the lines in are in the right order again).
4. Join the re-reversed chunks back together again into a single output file.


### Step 1: Start a Jupyter notebook

Create a "fresh" Jupyter notebook in an empty folder. It doesn't matter where; run it on your laptop, use Google colab, whatever you like.

### Step 2: Install Crossflow

In a fresh cell write:

```bash
!pip install crossflow
```

Hit shift-return and it should install within a few seconds.


### Step 3: Import Crossflow

Paste the line below into a fresh cell and run it, to import the first part of Crossflow you will need:

```python
from crossflow.tasks import SubprocessTask
```

### Step 3: Create a text file to process

Copy the little program below into a fresh cell and run it:

``` python
with open('test.txt', 'w') as f:
    for i in range(100):
        f.write(f'line {i}\n')
```

You should see a file 'test.txt' has appeared in the current directory.

### Step 4: tac or tail?

Depending on the flavour of Unix that's installed on the machine your Jupyter notebook is running on, the command to reverse the lines in a file is either *tac* or *tail -r*. Find out which you have. In a fresh cell run:

```bash
!which tac
```

If you get nothing back, you will need to use *tail -r*, otherwise you have *tac* available.

### Step 5a: create yoiur first task (*tac* version)

If you were working in a unix terminal and had a file called 'lines.txt' that you wanted to reverse, you would type:

```bash
tac lines.txt > reversed.txt
```

Use this prototype to create a Crossflow *task* that could do the same job. Paste the following lines into a fresh cell and run it (though there should be no output - that comes later):

```python
reverse = SubprocessTask('tac lines.txt > reversed.txt')
reverse.set_inputs(['lines.txt'])
reverse.set_outputs(['reversed.txt'])
```

Go on to step 6.

### Step 5b: create your first task (*tail -r* version)

If you were working in a unix terminal and had a file called 'lines.txt' that you wanted to reverse, you would type:

```bash
tail -r lines.txt > reversed.txt
```

Use this prototype to create a Crossflow *task* that could do the same job. Paste the following lines into a fresh cell and run it (though there should be no output - that comes later):

```python
reverse = SubprocessTask('tail -r lines.txt > reversed.txt')
reverse.set_inputs(['lines.txt'])
reverse.set_outputs(['reversed.txt'])
```

### Step 6: Test run your new task

Crossflow tasks are Python functions. Though you don't normally do this, you can run them interactively. In a new cell, run:

```python
reversed = reverse('test.txt')
print(type(reversed))
print(reversed)
```

Firstly notice that the file the function operated on was called 'test.txt', but the prototype used to create the task called it 'lines.txt'. This is because the filenames in the prototype are just placeholders for what the inputs and outputs should be; you could have defined the task using 'tac input > output', or 'tac x > y' and as long as you used the same ('input', 'output', or 'x', 'y') in the set_inputs() and set_outputs() lines you would be fine.

Secondly notice that the function returns a particular types of Python object - a crossflow *FileHandle*. If you *print()* it it looks like a string, but actually it's closer to a *Path* object, if you are familiar with these from Python's *pathlib* module.

Like a Python *Path*, a *FileHandle* has a *read_text()* method; give it a go:

```python
print(reversed.read_text())
```

If you want to save a copy of the content of your *FileHandle*, use its *save()* method:

```python
reversed.save('reversed.txt')
```

### Step 7: Clusters and Clients

While tasks can be run interactively, most usually they are performing some long-running or compute intensive work that you want to happen on some other computing resource. In Crossflow the computing resource is called a *cluster*, and your Python programme talks to it using a *client*.

For this simple example we will create a *cluster* which is just a separate process running on this same computer. We will create this using the *LocalCluster* class from *Dask.Distributed*. Here's the code, copy it into a fresh call and run:

```python
from distributed import LocalCluster
from crossflow.clients import Client

cluster = LocalCluster(n_workers=1)
client = Client(cluster)
```

### Step 8: Submit your first task to your cluster

*Clients* act a bit like job scheduling systems. Try this:

```python
reversed = client.submit(reverse, 'test.txt')
print(reversed)
```

Compare the syntax to the way you ran the same task in Step 6. The client's *submit()* method takes the task object as the first argument, and the task's arguments as the following ones (there is only one in this case). Notice that *reversed* is now not a *FileHandle*, it's a *Future* (for a *FileHandle*). *Futures* are like "promises" that, at some time in the future when the task has finished, the corresponding *FileHandle* object will be available.

### Step 9: Retrieve the resuts from the cluster

Check the status of your *Future*:

```python
print(reversed.status)
```

It should be 'finished'. So now you can retrieve the *FileHandle*, which currently is still sitting on the *cluster*:

```python
real_reversed = reversed.result()
print(real_reversed.read_text())
```

### Step 10: Add a second step to your workflow

The next step in the workflow is to split the reversed file into five chunks. In a terminal window you could do this using the *split* command, as long as you know the total number of lines in the file (100 in this case):

```bash
split -l 20 reversed.txt

```
This command would generate five files, called "xaa", "xab", "xac", "xad", and "xae", each with 20 lines of text.

Now see how you can create a *Task* to do the same job:

```python
split5 = SubprocessTask('split -l {chunk_size} infile')
split5.set_inputs(['infile', 'chunk_size'])
split5.set_outputs(['xaa', 'xab', 'xac', 'xad', 'xae'])
```

Notice a couple of things:

1. By default, *Crossflow* assumes that placeholders in task definition prototypes are filenames. Arguments like *chunk_size* are surrounded by braces ({}) to flag that they are not.

2. Notice that *set_outputs()* makes reference to files that are not even mentioned in the prototype - this is fine, as long as their names can be guaranteed, as they are here.

### Step 11: Run the second step

Here's the code for you - notice the way *chunk_size* is determined - you don't have to do it this way, but this approach is nice and compact:

```python
n_lines = reversed.result().read_text().count('\n')
chunks = client.submit(split5, reversed, n_lines//5)
print(chunks)
```

Notice that rather than pass the name of a file as the second argument, we don't even pass a *FileHandle* to it, we actually pass the *Future* to that *FileHandle*. This is one of the strengths of *Tasks*, as it means you can submit tasks to the *cluster* that might depend on data you haven't actually finished generating yet.

### Step 12: Introducing *map()*.

The next step in the workflow is to apply the *reverse* task we have already constructed to each of the chunks. One way to do this would be to write some code something like:

```python
revchunks = []
for chunk in chunks:
    revchunks.append(client.submit(reverse, chunk))
```

But this can be some more succinctly using the client's *map()* method (so copy and run this):

```python
revchunks = client.map(reverse, chunks)
```

The power of the *map()* method comes from the fact that *cluster* consists of a *scheduler* and one or more *workers*, and the scheduler will distribute the different tasks to different workers to balance out the workload. This has no effect in the current situation (remember you launched your cluster with just one worker), but on an HPC system you might be able to have tens or hundreds of workers...

### Step 13: Putting it back together

The final step in our workflow involves concatenating the reversed chunks back into a single output file. Here's the code:

```python
cat = SubprocessTask('cat chunk* > whole')
cat.set_inputs(['chunk*'])
cat.set_outputs(['whole'])
```

Notice that its very easy to build a task that takes a list of files as input. When the task is run, the corresponding argument should be list or tuple of filenames, filehandles or futures - which is what *revchunks* already is, so:

```python
whole = client.submit(cat, revchunks)
print(whole.result().read_text())
```

Copy and run these two chunks of code - hopefully the result is what you expect!

### Step 14: Putting it all together

Let's put the whole workflow together into a single Python function:

```python
def my_workflow(input_filename, output_filename):
    '''
    Run the whole workflow
    
    For compactness the function does not redefine the tasks, etc.
    '''
    reversed = client.submit(reverse, input_filename)
    n_lines = reversed.result().read_text().count('\n')
    chunks = client.submit(split5, reversed, n_lines//5)
    revchunks = client.map(reverse, chunks)
    whole = client.submit(cat, revchunks)
    whole.result().save(output_filename)
```

Does it work? As a small challenge, can you enhance the workflow so the number of chunks the file is split into can be chosen?


```python

```
