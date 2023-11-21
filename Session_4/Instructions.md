## Session 4: Analysis of the Path Sampling Workflow Results

This short session is an opportunity to explore the pathway between the "open" and "closed" states of adenylate kinate that your workflow has identified.

### Orientation

Navigate to the *session_4* folder and open a terminal window here, if you don't already have one.

In this folder you will find *pathtraj*, a little Python utility to post-process your workflow data. If you type:

```bash
./pathtraj -h
```
You will get a short help message. 

To run *pathtraj* you specify your two ensemble trajectory files, two reference "topology" files (but here, actually Gromacs .gro files), and the indices of the snapshots in the two ensembles that you want to find the shortest pathway between. The output is a multi-model PDB format file that you can download and visualise.

The "start" and "end" structures will be the first snapshots in each of the two ensemble trajectory files, so the command you need is:

```bash
./pathtraj ../Session_3/ensembleA.xtc ../Session_3/1ake_em.gro 0 ../Session_3/ensembleB.xtc ../Session_3/4ake_em.gro 0 pathway.pdb
```
The job may take a minute to run. 

When it has finished, you will see the file "pathway.pdb" in the file browser pane on the left. If you ctrl-click on this you get the option to download it to your laptop. Once there, you should be able to use your favourite visualisation program to see the transition path.


```python

```
