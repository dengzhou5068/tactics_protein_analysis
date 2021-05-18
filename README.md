# TACTICS Pocket Finder Code
This code finds the locations of possible cryptic pockets within MD trajectories.

## Requirements
Running TACTICS requires that the following be installed:
* The MDAnalysis python package.  Can be installed using `pip install --upgrade MDAnalysis` or (if you have conda) `conda config --add channels conda-forge && conda install mdanalysis && conda update mdanalysis`.  See https://www.mdanalysis.org/pages/installation_quick_start/.
* The scikit-learn python package, version 0.21.2.  Can be installed using `pip install scikit-learn==0.21.2`.  See https://scikit-learn.org/stable/install.html, but make sure to install the correct version!  The latest version of scikit-learn will not work with TACTICS.
* The pandas python package.  Can be installed using `pip install pandas`.  See https://pandas.pydata.org/docs/getting_started/install.html.
* Autodock Vina must be installed so that it can be run using the command `vina`.  See http://vina.scripps.edu/.  Adding a command like `export PATH=/path/to/vina/bin/:$PATH` to your `.bashrc` file should work (after downloading Vina).
* ConCavity must be installed so that it can be run using the command `concavity`.  See https://compbio.cs.princeton.edu/concavity/; download and compile the source code.  Then add something like `export PATH=/path/to/concavity/bin/x86_64/:$PATH` to your `.bashrc`.
* VMD must be installed so that it can be run using the command `vmd`.  See http://www.ks.uiuc.edu/Research/vmd/.
* PyMOL must be installed.  For info on open-source versions of PyMOL, see https://pymolwiki.org/index.php/Linux_Install and https://github.com/schrodinger/pymol-open-source.  Installing using a package manager ex. `apt-get` might be easier than installing from source.  Alternatively, see https://pymol.org/2/ for info on the proprietary version of PyMOL.
* MGLTools must be installed.  See https://ccsb.scripps.edu/mgltools/.  WARNING: As of February 25, 2021, MGLTools doesn't work on Mac OS Catalina or newer.  Use a virtual machine or a different computer.
* The file `get_dock_score.py` must be modified so that `mgltools_loc`, `pythonsh_loc`, and `prepare_receptor_loc` stores the locations of the MGLTools software.

## Usage
#### Locations of the Code
The directory `train_model` includes the code that was used to train the ML model.  It is expected that most software users will not need to run the code in this directory.

The directory `run_model` contains the code to run the model.   ***Users should run the code while `run_model` is the working directory.***

**Warning: the MD trajectory should be aligned to itself, so that the center of mass remains constant.  This matters because TACTICS finds the change in residue positions; motion of the entire protein would bias this.**

#### How to Run the Code

The algorithm is expected to be used through the function `tactics`.  See
`run_model/run_tactics.py` for an example.  Calling the function
`tactics` will run the pocket-finding algorithm and store the results
wherever `output_dir` is.

Here is the order of arguments to 'tactics`:

```
tactics(output_dir, apo_pdb_loc, psf_loc=None, dcd_loc=None, universe=None,
        num_clusters=None, alt_clustering_method=None, ml_score_thresh=0.8,
        ml_std_thresh=0.25, dock_extra_space=8, clust_max_dist=11)
```
Here is an explanation of each argument.  Note that either `universe` or both `psf_loc` and `dcd_loc` must be specified.

 * `output_dir` : string.  The name of the directory where the output is stored.  If the directory already exists, its contents will be overwritten.
 * `apo_pdb_loc` : string.  The path to the PDB file of the "apo" structure before MD has started.  This is compared with the frames from the MD trajectory.
 * `psf_loc` : string, optional.  The path to the MD trajectory's PSF file.   If `psf_loc` is `None`, then `dcd_loc` must be `None` and `universe` must not be `None`.
 * `dcd_loc` : string, optional.  The path to the MD trajectory's DCD file.  If `dcd_loc` is `None`, then `psf_loc` must be `None` and `universe` must not be `None`.
 * `universe` : MDAnalysis universe, optional.  An MDAnlysis universe with the protein conformational ensemble (ex. aligned MD trajectory).  If `universe` is `None`, then `psf_loc` and `dcd_loc` must not be `None`.  The code re-opens the file(s) in the universe.  So when the universe is initialized, the file paths must be specific enough that the code can find the files from the `tactics` function call.  Additionally, the universe's input structures must have segids of the form PROA, PROB, etc. where A,B act as the chain label.
 * `num_clusters` : int.  The number of k-means clusters to create.  Either `num_clusters` or `alt_clustering_method` must be `None`.
 * alt_clustering_method : MDAnalysis ClusteringMethod object.  An algorithm used to be used to cluster the trajectory.  If `alt_clustering_method` is `None`, then k-means will be used.  Either `alt_clustering_method` or `num_clusters` must be `None`.
 * `ml_score_thresh` : float, optional.  The ML confidence score threshold for determining if a residue is "high-scoring".  It must be between 0 and 1.  The default value is 0.8
 * `ml_std_thresh` : float, optional.  The algorithm ignores residues that have high ML confidence scores in all frames.  It does this by ignoring residues for which the standard deviation of the confidence scores among MD snapshots is less than ml_std_thresh.  This number must be between 0 and 1.  The default value is 0.25.
 * `dock_extra_space` : float, optional.  The space (in Angstroms) added to each side of the predicted site when determining the region to perform docking in.  The default value is 8.  This value is not expected to need changing from system to system; the default value is recommended.
 * `clust_max_dist` : float, optional.  The distance threshold (in Angstroms) to determine if a residue with a high ML score should be included in a cluster of other high-scoring residues.  The default value is 11.  This value is not expected to need changing from system to system; the default value is recommended.
 
 
#### What Files Are Created?
 
 
 `output_dir` contains many files.  Most of them are created by intermediate steps of the algorithm and aren't useful.  Here are the files that are expected to be useful:
 
  * `display_black_bg.pml` is a PyMOL script to display the results.  Recall that the function `tactics` begins by clustering the MD trajectory based on structure.  This PyMOL script displays a frame from each cluster; it does the following:
    * Residues that are predicted by ML to be in cryptic pockets are shown in white sticks.
    * Residues with high ML scores are clustered; fragment docking is done on each cluster.  The B-factor putty displays the fragment dock scores.
    * The boxes show the boundaries for the fragment dock calculations.  They are slightly larger than the predicted binding sites.
  * `display_white_bg.pml` is similar to `display_black_bg.pml`, except that the ML predictions are shown in black and the background is white.  The white background is often more suitable for making publication images than the black background.
  * Each cluster's structure is written to the file `centroid_<cluster_num>.pdb` where `<cluster_num>` is the number of the cluster.
  * `written_output.txt` lists each predicted site.  It records the center and size of each docking box (from when Autodock Vina was run within the algorithm); this can be used as an approximate quantification of the site's location.
  * `times.txt` lists the total time in seconds that TACTICS took to run, along with the times taken by several steps of the algorithm.
#### How Can the Output Be Interpreted?
First, change the working directory to whatever was passed as `output_dir`.  Then, run one of the PyMOL scripts.  The script displays all clusters.  This is necessary in order to get PyMOL to scale the b-factors correctly.  But it's hard to understand.  To examine the output, do the following:

 * Hide everything.
 * Display a single cluster, and the boxes for it.
 * Examine the box size.  If the box is too big, then the fragment docking will be inaccurate.  If the box is no bigger than a typical protein domain, then the docking should work.
     * If a box is too big, this means that the algorithm can't work on that part of the structure.  Ignore that area; hopefully the issue isn't present in all snapshots.
     * Occasionally, a small box will be partially or entirely within a larger box.  The overlap may impact the reliability of the dock scores; this should be considered when interpreting the code's output in this situation.
 * Examine the sticks (ML predictions) and b-factor putty (fragment docking).  Ideally, the residues with sticks should also have high b-factors.  This means that the ML prediction and docking agree.
     * If the sticks and the high b-factors are in different areas, then fragment docking didn't support the ML prediction.  This means the prediction is probably incorrect; there is probably no pocket.
 * It may be useful to load `centroid_<cluster_num>.pdb` and display it as a cartoon.  This will make it easier to determine where in the protein each predicted pocket is.
 * Repeat the process for each cluster centroid.

## Debugging
If the computer runs out of RAM, it may stop running the code and give the error message `Killed`.  If this happens, reduce the size of the input file or free up more RAM.

If the code predicts numerous pockets but each pocket only has one residue, then the segids of the input may be wrong.  They must be of the form `PROA`, `PROB`, etc.

If the code predicts no pockets, then `ml_score_thresh` and `ml_std_thresh` may be too high.

If the code gives the error message `KeyError: '1:A'`, then the apo structure's residue and chain nomenclature may not match the nomenclature used in the trajectory.
