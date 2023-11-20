## Session 5: An Iterative Workflow for Docking-based Ligand Optimisation

This example demonstrates the application of Crossflow to a very different type of workflow from the last. In particular this example shows how to include your own command line driven tools, and some more advanced capabilities in *Crossflow* *Task* construction.


### Orientation

When you have navigated to the `session_5` folder via the file browser pane on the left of your JupyterLab window, you will see the following files:

* *3atl_receptor.pdbqt*: The structure of trypsin in "pdbqt" format, ready for docking.
* *3atl.conf*: A configuration file for AutoDock Vina

Open a terminal in this folder, and check out the command line tools you will be using. You will be using *AutoDock Vina* for the docking calculations. This is already installed for you on Archer2, to check this type:
```bash
vina --help
```
You should get a detailed help message.

To run *AutoDock Vina* ("*vina*") you need three input files: a receptor structure, a ligand structure, and a configuration file with docking parameters. The ligand and receptor files must be in *pdbqt* format. The receptor file and configuration file are already prepared for you, but you need to create the ligand file. You will use *Open Babel* for this, which also is already installed, to check this type:
```bash
obabel -H
```
You should get a detailed help message. So make a .pdbqt file for our lead ligand, which is phenyl acetate. The SMILES string for this is "CC(=O)Oc1ccccc1":
```bash
obabel -:"CC(=O)Oc1ccccc1" -ismi -O phenyl_acetate.pdbqt -opdbqt --gen3d
```
Now you can run *vina*:
```bash
vina --ligand phenyl_acetate.pdbqt --receptor 3atl_receptor.pdbqt --config 3atl.conf --out dock_result.pdbqt
```
The run should only take a few seconds, at the end of which the docking score is reported, and the file dock_result.pdbqt created. The file contains the ligand in its docked pose, but for most of the workflow it will only be the docking score that interests us.

The final command line tool you will be using is *g_sim.py*; this is a "home grown" Python script that searches the *GuacaMol* database for molecules similar to a query, provided as a SMILES string. Check it out:
```bash
./g_sim.py -h
```
You should get a brief help message.

To demonstrate it in action, let's search the *GuacaMol* database for structures similar to phenyl acetate:
```bash
./g_sim.py 'CC(=O)Oc1ccccc1' g_data.pkl -n 10
```
After a few seconds the ten molecules in the database most similar to the query are listed. Notice that the top one is the query itself - it turns out that phenyl acetate is in *GuacaMol*. The others are structurally similar, so might also bind to trypsin - they might even bind more strongly. You can see that you have all the components of a an iterative workflow here:

1. Turn each of the similar molecules (except phenyl acetate - you have already docked that) from SMILES format into pdbqt format.
2. Dock each of them against the receptor.
2. Add the structure and docking score of each to a table of results.
3. Scan the table to find the ligand with the highest docking score.
4. Search the *GuacaMol* database to find similar molecules.
5. Check the returned list against those in the table of results so far - no point in redocking ones you have already tested.
6. If there are novel 'untested' molecules remaining, convert them to pdbqt format and go to step 2, otherwise your search is over, your optimal molecule is in your table of results.

### Step 1: Build a Crossflow Cluster for your Workflow

This starts off nearly the same as in the previous example, so here is all the code you need:
```python
from crossflow.tasks import SubprocessTask
from crossflow.clients import Client
from crossflow.filehandling import FileHandler
from dask_jobqueue import SLURMCluster
from distributed import as_completed, wait

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
                       job_script_prologue=['module use /work/e280/shared/hecbiosim-software/modules',
                                            'module load autodock-vina-1.2.5',
                                            'module load openbabel-3.1.1',
                                  'export OMP_NUM_THREADS=1',
                                  'source /work/e280/e280-workshop/<username>/myvenv/bin/activate'])
```
The main difference is in the *job_script_prologue* where we load some local modules that provide *AutoDock Vina* and *Open Babel*.

### Step 2: Create the *Tasks*

Now create *Tasks* for *vina*, *obabel* and *g_sim.py*:
```python
vina = SubprocessTask('vina --ligand lig.pdbqt --receptor receptor.pdbqt --config  config.conf --out output.dat')
vina.set_inputs(['lig.pdbqt', 'receptor.pdbqt', 'config.conf'])
vina.set_outputs(['output.dat'])

smi2pdbqt = SubprocessTask('obabel -:"{smiles}" -ismi -opdbqt -O x.pdbqt --gen3d')
smi2pdbqt.set_inputs(['smiles'])
smi2pdbqt.set_outputs(['x.pdbqt'])

g_sim = SubprocessTask('/work/e280/shared/GuacaMol/g_sim.py "{smiles}" /work/e280/shared/GuacaMol/g_data.pkl -n 10 > similars.smi')
g_sim.set_inputs(['smiles'])
g_sim.set_outputs(['similars.smi'])
```
A couple of things to notice:

* Both *obabel* and *g_sim.py* commands take arguments that are text strings, not the names of files. To make this happen, the job templates show these arguments in braces ({}). 
* Because the *g_sim.py* command is in a shared directory that is not on our PATH (unlike *vina* and *obabel*), we have specified the absolute paths to both the executable and the data file. Notice this only works because the workers will be able to see this shared directory too - you would need a different approach if, for example, you were using a *cluster* that was a collection of instances in the cloud, where there was no shared file system.

Thinking about the *vina* *Task*, two of the arguments - the receptor file and the configuration file - are always going to be the same. *Crossflow* *Tasks* can specify arguments as *constants* as part of the *Task* creation process, and then do not appear when it is run:
```python
fh = FileHandler()
receptor = fh.load('3atl_receptor.pdbqt')
config = fh.load('3atl.conf')
vina.set_constant('receptor.pdbqt', receptor)
vina.set_constant('config.conf', config)
```
Copy this block into a fresh cell and run it - there should be no output.

Now when we run the *vina* *Task*, instead of being:
```
dock_result = vina(ligand, receptor, configuration)
```
it's just:
```
dock_result = vina(ligand)
```

### Step 3: Processing the Docking Results

When the docking jobs finish, we will need to extract the docking score from the output files and store it somewhere. A simple dictionary, keyed by the SMILES string for the ligand will do for this. Parsing docking results files for the docking score is quite simple, as long as the job completes succesfully, however there is a non-negligible possibility that it won't - in this case, mainly because the conversion of the ligand structure from SMILES format to pdbqt format by *obabel* can be buggy for "unusual" structures. The code snippet here demonstrates how we can deal with all this:

```python
def extract_score(future):
    wait(future) # Wait for the job to end
    if future.status == 'finished': # This status means the job ran OK
    # The docking score appears in the file immediately after the string "RESULT":
        words = future.result().read_text().split()
        for i in range(len(words) - 1):
            if words[i] == 'RESULT:':
                return float(words[i+1])
    else: # if something has gone wrong, just return a large, bad, score
        return 1000.0
```

Copy and paste this into a new cell and run it - there should be no output.

### Step 4: Test the Workflow Components

Now we can step through one iteration of the workflow to test out the components we have created.

```python
cluster.scale(4) # 4 workers will do for now - we can explore this later
client = Client(cluster)
results = {} # An empty dictionary, ready to store docking results
lead_molecule = "CC(=O)Oc1ccccc1" # phenyl acetate
lead_pdbqt = client.submit(smi2pdbqt, lead_molecule) # convert from SMILES to pdbqt
dock = client.submit(vina, lead_pdbqt) # Dock it
results[lead_molecule] = extract_score(dock) # Extract the docking score and store
print(results)
```
Copy this into a fresh cell and run - after some time, the docking score should be printed.

Now lets look at the search of *GuacaMol* for similar structures to our current lead:

```python
similars = client.submit(g_sim, lead_molecule)
similar_smis = similars.result().read_text().split('\n')[:-1]
new_candidates = []
for s in similar_smis:
    if not s in results:
        new_candidates.append(s)
print(new_candidates)
```
Line 2 of this snippet is turning the contents of the *file* returned by *g_sim.py* (a list of SMILES strings, one per line) into a Python list of *strings* for the other tasks in the workflow. The last section makes sure that any molecules found that are ones we have already docked in some previous iteration, are removed.

Again, copy this into a fresh cell and run - after some time, the list of similar molecules in *GuacaMol* should be printed.

### Step 5: Construct and Run the Full Workflow

We can now put all the pieces together:

```python
icycle = 1
while len(new_candidates) > 0:
    print(f'Cycle {icycle}: docking {len(new_candidates)} candidates')
    # Convert the ligands to pdbqt format and dock, all in parallel:
    pdbqts = client.map(smi2pdbqt, new_candidates)
    dock_results = client.map(vina, pdbqts)
    # As the docking results come through, extract the scores data:
    for f in as_completed(dock_results):
        k = dock_results.index(f)
        results[new_candidates[k]] = extract_score(f)
    # Sort the dictionary of results so the best (most negative) score is first:
    results = {k:v for k,v in sorted(results.items(), key=lambda item: item[1])}
    lead_molecule = list(results.keys())[0]
    print(f'Current lead molecule: {lead_molecule}, docking score: {results[lead_molecule]}')
    # Find similar molecules:
    similars = client.submit(g_sim, lead_molecule)
    similar_smis = similars.result().read_text().split('\n')[:-1]
    new_candidates = []
    for s in similar_smis:
        if not s in results:
            new_candidates.append(s)
    if len(new_candidates) == 0:
        print('No new candidates, search completed')
    icycle += 1
```

Copy into a new cell and run. There is an element of randomness in docking, so you may not get exactly the same results each time, but you should see each iteration finding a molecule with a somewhat better docking score than any found before, until the search fails to find novel molecules to dock.

### Step 6: Visualise the Results

Let's take a look at our optimised ligand. We will re-run the docking calculation for it and this time save the result into a file which can later be downloaded to your own laptop and visualised (if you have Chimera installed):
```python
optimum_pdbqt = client.submit(smi2pdbqt, lead_molecule)
dock_pose = client.submit(vina, optimum_pdbqt)
dock_pose.result().save('optimised_ligand.pdbqt')
```
You will see a new file "optimised_ligand.pdbqt" appear in the browser pane on the left. If you ctrl-click on this, you will see an option to download it. You can download a copy of the receptor .pdbqt file in the same way.

We can also use the Python rdkit package to visualise the best molecules. Here is the code:
```python
from rdkit import Chem
from rdkit.Chem import Draw

top_8 = list(results.keys())[:8] # Best 8 ligands - you could change this
# Convert to RDKit molecule objects:
ms = [Chem.MolFromSmiles(k) for k in top_8]
# Create an image of them showing structure and docking score:
img=Draw.MolsToGridImage(ms, molsPerRow=4, subImgSize=(200,200), legends=[str(results[k]) for k in top_8])
img
```

### Next...

* How does performance scale with the number of workers?
* What happens if you start from a similar but different lead molecule?
* What happens if you search *GuacaMol* for a greater number of similar ligands each iteration (hint: you will need to edit the definition of the *g_sim* *Task*)
