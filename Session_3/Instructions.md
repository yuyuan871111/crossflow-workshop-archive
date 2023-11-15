## Coding a Conformational Transition Workflow with Crossflow

In this session we will step through the process of coding an enhanced sampling workflow to map the conformational transition of adenylate kinase between the "closed" and "open" conformations.

### Step 1: Orientation

In your Jupyterlab window, navigate to the *Session_3* folder. There you will find six files:
* *1ake_em.gro* and *1ake.top*: Gromacs coordinate and topology files respectively for the closed conformational state.
* *4ake_em.gro* and *4ake.top*: the same for the open state. 
* *nvt.mdp*: an input file that defines parameters for a short MD simulation.
* *slurmscript.sh*: a Slurm job submission file.

### Step 2: Run a test MD job

Before we start coding up a workflow that will end up running large numbers of MD simulations, let's make sure we can run one the "normal" way.

From your Jupyterlab window start a terminal session, and then open the file *slurmscript.sh* in an editor of your choice. If you have used Gromacs on Archer2 before, it should look fairly familiar. Key points to note are:

* For this workshop jobs will use the "e280-workshop_1018917" partition (`-p` option).
* Jobs will be charged to the "e280-workshop" account (`-a` option).
* Jobs will use the "reservation" quality of service parameter (`-q` option).

As usual, runing a Gromacs MD job is a 2-part process; first the coordinate, parameter and job definition files are processed by *grompp* to give a "portable run file" (in this case, *1ake_nvt.tpr*), then the *-deffnm* argument to the following *mdrun* command defines that all input and output files for the MD run itself will have a base filename of *1ake_nvt*. Finally this script also includes a post-processing step: the Gromacs *trjconv* utility is used to post-process the trajectory file to remove artifacts due to the use of periodic boundary conditions. Notice on this line how we get round the issue that *trjconv* is usually run interactively, and asks for user input (which group to process).

Once you have satified yourself that the slurmscript is correct, submit the job in the usual way:
```bash
sbatch slurmscript.sh
```
Use *squeue -u \<username\>* to follow the job; because we have a reservation on Archer2 for this workshop it should start almost straight away, and finish in less than a minute.

You will see the job generates a set of output files, all with names that begin *1ake_nvt*, with the extension defining the file type according to the usual Gromacs conventions.

### Step 3: Build a Crossflow Cluster to Run the Same Job

Now let's see how we can run the same MD simulation workflow, but via Crossflow.

Start a new Jupyter notebook, and copy the following into the first cell:

```python
from crossflow.tasks import SubprocessTask
from crossflow.clients import Client
from dask_jobqueue import SLURMCluster
```

The first two lines should be familiar from the earler session. In that we connected a crssflow *client* to a *cluster* that was actually another process running on the same machine, using the *LocalCluster* class. In this case in line 3 we import the *SLURMCluster* class from the *dask_jobqueue* package, which will allow us to create clusters of (maybe many) workers that are individual Archer2 nodes.

Run this cell (there should be no output), then copy the following into the cell below:

```python
cluster = SLURMCluster(cores=1, 
                       job_cpu=1,
                       processes=1,
                       memory='256GB',
                       queue='e280-workshop_1018917',
                       job_directives_skip=['--mem', '-n '],
                       interface='hsn0',
                       job_extra_directives=['--nodes=1', 
                           '--qos="reservation"', 
                           '--tasks-per-node=128'],
                       python='python',
                       account='e280-workshop',
                       walltime="06:00:00",
                       shebang="#!/bin/bash --login",
                       local_directory='$PWD',
                       job_script_prologue=['module load gromacs',
                                  'export OMP_NUM_THREADS=1',
                                  'source /work/e280/e280/<username>/myvenv/bin/activate'])
```
There is a lot to look at here; most of it is "boilerplate" and you should take a look at the *dask_jobqueue* [documentation](here) if you want a full explanation, but for now we just pull out a few key features:

* the first three arguments (`cores`, `job_cpu`, and `processes`) ensure that each worker starts on, and has full access to, a full node on Archer2.
* The `interface="hsn0"` argument ensures that communication between workers in the cluster will use the system's fast interconnect.
* The list passed to the `job_script_prologue` argument is where commands that set up the computing environment are added. **Note you will need to edit this to insert your correct username**.

Once edited, run the cell. It should run fast, as all that happens here is that the configuration of the cluster is defined - no computing resources are generated at this point.

That happens next: copy the next two lines into a fresh cell and run it:

```python
client = Client(cluster)
cluster.scale(1) # start a single worker
```
After a short time you should see a logfile for a slurm job appear in the file browser pane on the left of your JupyterLab window - the worker has been launched and is now ready to receive tasks.

### Step 4: Build and Execute the Crossflow Tasks to Run the Same Job

From the file *slurmscript.sh* we can work out the template strings required to create Crossflow *Task* that would run an equivalent job:

```python
grompp = SubprocessTask('gmx grompp -f x.mdp -c x.gro -p x.top -o system.tpr -maxwarn 1')
grompp.set_inputs(['x.mdp', 'x.gro', 'x.top'])
grompp.set_outputs(['system.tpr'])

mdrun = SubprocessTask('srun --distribution=block:block --hint=nomultithread gmx_mpi mdrun  -deffnm system')
mdrun.set_inputs(['system.tpr'])
mdrun.set_outputs(['system.log', 'system.trr'])

makewhole = SubprocessTask('echo 0 | gmx trjconv -f broken.trr -s system.tpr -o whole.trr -pbc whole')
makewhole.set_inputs(['broken.trr', 'system.tpr'])
makewhole.set_outputs(['whole.trr'])
```

This creates a *Task* called *grompp* that takes three input arguments - an mdp file, a coordinates file, and a topology file - and returns one output: a tpr file, a second *Task* called *mdrun* that takes a tpr file as input and returns two output files: a log file and a trajectory file, and a final *Task* called *makewhole* that takes a trajectory file and tpr file as input, and returns a new trajectory file.

Notice that the *mdrun* task construction ignores many of the output files that the MD run will generate (e.g. the checkpoint file and energy info file). This is the normal way of working with *Crossflow* - whatever the underlying command line tool might do, the *Task* is specified in whatever way is needed to return just the information of interest, all other outputs are discarded.

Copy these lines into a fresh cell and execute it (there should be no output).

Now we are ready to run the job. This runs MD from the first "end" of the pathway we want to explore, we will label this "A". Here's the code, put it into a fresh cell and run it:

```python
tprA = client.submit(grompp, 'nvt.mdp', '1ake_em.gro', '1ake.top')
logA, trajA = client.submit(mdrun, tprA)
wholeA = client.submit(makewhole, trajA, tprA)
print(logA)
print(trajA)
print(wholeA)
```

When you run this cell you will see that *logA*, *trajA* and *wholeA* are *futures* with a status of *pending* - the job has not finished yet.

Copy the three *print()* statements above into a fresh cell, wait a few seconds and then run it. Has the status of the futures changed? If not, wait a bit and run again, until they show as "finished".

If you look in the file browser pane you will see no new files seem to have been generated. This is because *crossflow* tasks are run in temporary "sandbox" folders, and information is only returned as *filehandles*. If you want to generate "real" output files, you need to make use of each filehandle's *.save()* method:

```python
logA.result().save('1ake_test.log')
wholeA.result().save('1ake_test.trr')
```

Copy this into a new cell and run it, and convince yourself that the file *1ake_test.log* matches *1ake_nvt.log* you generated in step 2.

### Interlude: Explore scaling and performance

To recap, you run *Task*s on the *cluster* via the *client*'s *.submit()* method, or * *map()* method if you have many jobs you can (potentially) run in parallel. You also know that the results from *submit()* and *map()* calls are returned as *Future*s for *FileHandle*s. Input arguments in these calls can also be *Future*s or *FileHandle*s, but for convenience they can also just be the names of files (i.e., *Path*s). *Crossflow* jobs generally run more efficiently if the inputs are not *Path*s, so to explore performance we will preprocess the starting coordinate files, topology files and mdp file into *FileHandles* in advance. To do this we create an instance of a *Crossflow* *FileHandler*; here is the code:
```python
from crossflow.filehandling import FileHandler
fh = FileHandler()
startcrdsA = fh.load('1ake_em.gro')
topA = fh.load('1ake.top')
mdp = fh.load('nvt.mdp')
```

Copy this code into a new cell and run it - there should be no output.

Now we will run the workflow again, but this time add some instrumentation so we can track the progress of the different tasks. We will also not just process the starting coordinates once, but run the workflow ten times, i.e. running ten independent short MD simulations starting from the same structure - for this we will use the *map()* method. The code is here:

```python
from distributed import wait, as_completed
n_reps = 10
startcrdsAs = [startcrdsA] * n_reps # replicate the starting structure
tprAs = client.map(grompp, mdp, startcrdsAs, topA)
logAs, trajAs = client.map(mdrun, tprAs)
wholeAs = client.map(makewhole, trajAs)
for future in as_completed(tprAs + trajAs + wholeAs):
    if future in tprAs:
        print(f'grompp job {tprAs.index(future)} completed')
    elif future in trajAs:
        print(f'mdrun job {trajAs.index(future)} completed')
    else:
        print(f'make_whole job {wholeAs.index(future)} completed')
```
Copy this code into a fresh cell and run it. Notice how the different tasks end up being scheduled on the cluster - which currently only has one worker (i.e., one Archer2 node).

Once it has finished, copy this code into a fresh cell and run it to scale up your cluster to 5 workers:
```python
cluster.scale(5)
```
if you look at the file browser pane on the left, you will soon see new slurm log files appearing, as the jobs to run the extra workers are launched.

Now go back to the previous cell and re-run it, to see how the extra workers mean the tasks can get processed in parallel.

Each of you has up to ten workers available to you - so spend some time exploring how scaling the cluster speeds up the workflow.

### Step 5: Run an MD Step from the 4AKE Endpoint

Back to our path sampling workflow. Using the same approach, we can generate our first MD trajectory that begins from the other end ("B") of the pathway we want to uncover:

```python
startcrdsB = fh.load('4ake_em.gro')
topB = fh.load('4ake.top')

cluster.scale(4) # Reset the size of the cluster to a moderate value
tprB = client.submit(grompp, mdp, startcrdsB, topB)
logB, trajB = client.submit(mdrun, tprB)
wholeB = client.submit(makewhole, trajB, tprB)
```
Copy this into a new cell and run it, but rather than wait for the job to complete, we can begin to consider the next step in the workflow.

### Step 6: Coding up the RMSD Calculation Step

If you look back at the workflow diagram presented in the session introduction, you will see that the next step will be to load the snapshots from both of the newly-run MD simulations into trajectory databases, one for each end of the path, and then calculate the full matrix of RMSD values between the two.

There is no Gromacs command that will calculate the RMSD of every snapshot in one trajectory file from every snapshot in another, so we are going to turn our attention to Python packages to do this - *MDTraj* to import the trajectory data, and *MDPlus* to perform the RMSD calculation using a "trick" that makes this much faster.

The discussion here will not go into the details of how *MDTraj* and *MDPlus* work, please refer to the online documentation for that if you are unfamilar with these packages.

We begin by creating two *MDTraj* *Trajectory* objects, one for snapshots generated from each "end" of our conformational transition (we are calling the objects "ensembles" because ultimately they will hold data from many different MD simulations):

```python
import mdtraj as mdt

ensembleA = mdt.load(wholeA.result(), top=startcrdsA)
ensembleB = mdt.load(wholeB.result(), top=startcrdsB)
```

Notice here how we pass *filehandles* for the trajectory and topology/coordinate files to *MDTraj*'s *load()* command - this works because *filehandles* are "pathlike".

Copy and paste this code into a fresh cell in your notebook and run it - there should be no output, but it may take a little time to run - this code is actually running in your notebook, not on the Crossflow cluster.


Doing an all-against-all RMSD calculation is computationally expensive, but there is a "trick" you can use to do it much faster - if slightly approximately - using Principal Component Analysis. The method involves performing PCA analysis to tranform both trajectories into the same PC space, then calculating the Cartesian distance between points in this space. This number, divided by the square root of the number of atoms, in general is a very close approximation to the RMSD. The code is here:

```python
import numpy as np
from mdplus.pca import PCA
from scipy.spatial.distance import cdist

def rmsd2(ensembleA, ensembleB, sel):
    """
    Calculate approximate 2D RMSD matrix via PCA

    Args:

        ensembleA (mdtraj trajectory): ensemble A of nA frames
        ensembleB (mdtraj trajectory): ensemble B of nB frames
        sel (string): mdtraj selection specifier for RMSD calculation

    Returns:

        rmsd2d (numpy array[nA, nB]): RMSD matrix
    """
    idxA = ensembleA.topology.select(sel)
    idxB = ensembleB.topology.select(sel)
    x = np.concatenate([ensembleA.xyz[:,idxA], ensembleB.xyz[:, idxB]])
    p = PCA()
    scores = p.fit_transform(x)
    d = cdist(scores[:len(ensembleA)], scores[len(ensembleA):])
    return d / np.sqrt(len(idxA))
```

Copy and paste this code into a fresh cell in your notebook and run it - there should be no output.

### Step 7: Coding up and Running the Pair Selection Step

Again, looking back at the workflow diagram you will see the next step involves extracting the *N* shortest distances (RMSDs) between structures in the two ensembles. There are many possible approaches to this, the one we use here puts the shortests pairs into a dictionary structure:

```python
def get_pair_distances(ensembleA, ensembleB, selection='name CA', max_pairs=10):
    '''
    Calculate the RMSD distances between all snapshots in each of two ensembles and return the closest

    Args:
       ensembleA (mdtraj trajectory): first ensemble
       ensembleB (mdtraj trajectory): second ensemble
       selection (string): mdtraj selection for RMSD calculation
       max_pairs (int): number of closest pairs to return

    Returns:
       pairlist: sorted dictionary with tuple of snaphot indices as keys, RMSDs as values
    '''
    pairdist = {}
    d = rmsd2(ensembleA, ensembleB, selection)
    for i in range(ensembleA.n_frames):
        for j in range(ensembleB.n_frames):
            key = (i, j)
            pairdist[key] = d[i, j]
        
    # sort by increasing RMSD:
    pairdist = {k:v for k, v in sorted(pairdist.items(), key=lambda i: i[1])[:max_pairs]}
    return pairdist

# Now run it:
closest_pairs = get_pair_distances(ensembleA, ensembleB, max_pairs=5)
print(closest_pairs)
```
Copy and paste this code into a fresh cell in your notebook and run it - you should end up seeing the five shortest distances (RMSDs) between structures in ensembleA and structures in ensembleB (the RMSD calculation being done just over Calpha atoms).

### Step 8: The Complete Workflow

We are ready to run the complete workflow now. Each iteration will involve:

1. Running an MD simulation on structures from each end of the *N* shortest inter-ensemble distances.
2. Processing the trajectories to make them "whole"
3. Adding the new structures to the growing ensembles "A" and "B".
4. Updating the list of *N* shortest inter-ensemble distances
5. Stopping if the the shortest distance is less than some threshold value, or going back to step 1.

Here is the code:

```python
max_cycles = 10 # Maximum number of workflow iterations
min_rmsd = 0.2 # Target minimum RMSD between structures from each ensemble
max_pairs = 5 # Number of shortest inter-ensemble pairs to take forward to next iteration
cluster.scale(max_pairs) # Scale the SLURMCluster up to max_pairs workers

for icycle in range(max_cycles):
    print(f'Starting cycle {icycle}...')
    shortestA = [k[0] for k in list(closest_pairs.keys())] # indices of chosen structures from A
    shortestB = [k[1] for k in list(closest_pairs.keys())] # ditto for B
    tprAs = client.map(grompp, mdp, [ensembleA[i] for i in shortestA], topA) # run grompp jobs in parallel
    tprBs = client.map(grompp, mdp, [ensembleB[i] for i in shortestB], topB) # ditto
    logAs, trajAs = client.map(mdrun, tprAs) # run MD jobs in parallel
    logBs, trajBs = client.map(mdrun, tprBs) # ditto
    wholeAs = client.map(makewhole, trajAs, tprAs) # remove PBC artifacts
    wholeBs = client.map(makewhole, trajBs, tprBs)
    wait(wholeBs) # Block here until all jobs are done
    print('MD runs finished, finding closest pairs...')
    ensembleA += mdt.load([t.result() for t in wholeAs], top='1ake_em.gro') # Add to ensembles
    ensembleB += mdt.load([t.result() for t in wholeBs], top='4ake_em.gro')
    closest_pairs = get_pair_distances(ensembleA, ensembleB, max_pairs=max_pairs)
    print(f'Cycle {icycle}: closest pairs = {closest_pairs}')
    if list(closest_pairs.values())[0] < min_rmsd: # The two ends of the path have "met" - stop.
        break
        
print('Path search completed')
cluster.scale(0) # Scale the cluster back to having no workers at all.
ensembleA.save('ensembleA.xtc') # Save the ensembles in Gromacs xtc format for later analysis.
ensembleB.save('ensembleB.xtc')
```

Copy this into a fresh cell and run. This is going to take some time, but as each iteration finishes you should see the shortest distance between sytuctures in the two ensembles gradually reduce, until it reaches 0.2 nanometyers, when the search will stop, and the ensembles generated from each start-point will be saved to disk in Groams .xtc format. In the next session we will take a look at the results.



```python

```
