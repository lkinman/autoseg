# SIREn (*S*ubunit *I*nference from *R*eal-space *En*sembles)
SIREn is a toolkit for analyzing heterogeneity in 3D volume ensembles generated via cryo-EM. SIREn performs automated detection of structural blocks to allow model-free detection of heterogeneity, and additionally includes a 3D Convolutional Neural Network (CNN) to predict binarization thresholds for 3D density maps. The 3D CNN can be used beyond the SIREn workflow to predict binarization thresholds for maps generated from either homogeneous or heterogeneous cryo-EM reconstruction tools. Structural blocks generated by SIREn can furthermore be used as input to [MAVEn](https://www.github.com/lkinman/MAVEn), which quantifies occupancy of structural features across a volume ensembles and permits isolation of particles bearing a feature of interest. 
  
## Literature


## Installing  
First create a conda environment for SIREn, as shown below:
```
conda create --name siren python=3.10
conda activate siren
conda install pandas jupyterlab matplotlib scipy
conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia
pip install networkx 3
pip install mrcfile
pip install natsort
pip install torch==2.0
```
Note that the appropriate version of torch and pytorch to install will depend on your system setup; make sure your torch version is compatible with your CUDA drivers. You can check your CUDA version with ```nvidia-smi```.

It is recommended that you install ChimeraX (install instructions can be found [here](https://www.cgl.ucsf.edu/chimera/download.html)) in order to visualize the structural blocks resulting from your SIREn analysis.

To install the SIREn code, simply git clone the source code:
```
git clone https://github.com/lkinman/SIREn.git
cd SIREn
python setup.py install
tar -xzvf weights.tar.gz
```

## Inputs
To run SIREn, you need a large ensemble of 3D density maps. These 3D density maps can come from any of a number of upstream processing softwares, but must be the same boxsize and aligned to a common reference frame. Additionally, the volumes must be downsampled -- we typically use a boxsize of 64, and other boxsizes have not been rigorously tested. Note that the box size SIREn can tolerate will also depend on what fraction of the box is occupied, as background voxels are masked in the first steps of SIREn. Large boxsizes will lead to combinatorial explosions in required computational time/power. Downsampling can be carried out by the scripts we provide to preprocess the volumes for the 3D CNN, as described below. 

You will also need to supply a threshold for binarization of the maps. We have trained a 3D Convolutional Neural Network (CNN) to predict a binarization threshold for each map. The CNN was trained on >3,000 maps (box size 64) with the accompanying recommended contour levels deposited on the Electron Microscopy Data Bank (EMDB). The script used to train our model is provided as 10262023_model_training.py, and the weights are provided as weights.pth. This model can be used to predict the binarization thresholds for 3D density maps generated by either homogeneous or heterogeneous cryo-EM reconstruction tools; we have evaluated the performance of the model on volumes generated by RELION, cryoSPARC, cisTEM, cryoDRGN, tomoDRGN, and 3DVA. 

The 3D CNN takes as input normalized, box size 64 volumes; users can prepare their volumes for input to the CNN by running ```siren preprocess```. The provided volumes can be either at box size 64 or full box size; if they are not at box size 64, ```siren preprocess``` will automatically downsample them, and write out the downsampled volumes. Note that the predictions generated by the model can also be denormalized to generate contour level predictions at either box size 64 or at the full box size. A ChimeraX plugin is available [here](https://github.com/mariacarreira/calc_level_ChimeraX) to predict and visualize the predicted isosurface levels for 3D density maps.

Lastly, we note that the volume ensemble must be large for successful deployment of SIREn. We typically use at least 200+ volumes, although the appropriate number of volumes will vary depending on the particle stack size. This approach relies on the statistical power of sampling large numbers of volumes, so using smaller ensembles will result in decreased sensitivity.

## Outputs
```siren eval_model``` outputs a predictions.csv file containing the relevant predictions (‘denormalized_predictions’) for each volume on which the model was evaluated. After generating predicted thresholds, we recommend inspecting the quality of the predictions both visually (e.g., using ChimeraX) and by comparing them with ground truth annotations for some maps. This functionality is directly implemented in ```siren eval_model```, which will provide a summary plot comparing predictions to annotated labels if labels are provided. 

```siren sketch_communities``` and ```siren expand_communities``` output a dictionary of seed blocks and final expanded blocks, respectively, where key-value pairs consist of each block's identifying number and constituent voxels. Additionally, the scripts automatically write out individual .mrc files for each block. These .mrc files can be visualized in ChimeraX. The .mrc files can additionally be supplied to [MAVEn](https://www.github.com/lkinman/MAVEn) to act as the input masks for ```calc_occupancy.py```.

## Using  
This software comprises 4 scripts and an additional interactive Jupyter notebook. The scripts:  


**1) Downsample and normalize maps for input to 3D CNN**



The CNN was trained on EMDB maps downsampled to box size 64 and normalized using the minimum and maximum voxel intensities so that voxel intensities lie between -1 and 1 for each map [https://doi.org/10.1186/s12859-022-04942-1]. The corresponding recommended contour levels are normalized accordingly for each map. As such, input maps need to be downsampled to box size 64 and normalized. The minimum and maximum voxel intensities are recorded for each map and saved in raw_map_stats.csv (for the full box volume) or downsampled_map_stats.csv (for the box 64 volume) to facilitate downstream denormalization of predicted labels. 

Note that volumes may be provided at box size 64 or full box size. If they are provided at full box size, downsampled volumes will be written out to outdir_downsampled. If users are providing maps from cryoDRGN or tomoDRGN, to save computational time and power, we recommend simply generating the maps at box size 64, rather than generating at full box size and then downsampling. 

```
siren preprocess --help

optional arguments:
  -h, --help            show this help message and exit
  --voldir VOLDIR        Path to input volume (.mrc) or directory containing volumes
  --labels LABELS        User-annotated labels for downsampled (non-normalized) volumes for normalization
  --outdir OUTDIR        Path to output directory for normalized volumes
  --outdir_downsampled OUTDIR_DOWNSAMPLED
                        Path to output directory for downsampled volumes
```  
e.g.
  
```
siren preprocess --voldir reconstruct_000000 --outdir cnn_outputs/
```  

**2) Predict the binarization threshold for each map, and assess the quality of model predictions on provided volumes**

```
siren eval_model --help

optional arguments:
  -h, --help            show this help message and exit
  --voldir VOLDIR        Path to input volume (.mrc) or directory containing normalized volumes
  --normalize_csv NORMALIZE_CSV
                        map_stats.csv (either downsampled or raw map stats)
  --labels LABELS        User-annotated labels for downsampled (non-normalized) volumes for evaluating model performance
  --weights_file WEIGHTS_FILE
                        Path to model weights (weights.pth or fine_tuned_weights.pth)
  --batch_size BATCH_SIZE
                        Minibatch size
  --outdir OUTDIR        Path to output directory
```  
  
e.g.
  
```
siren eval_model --voldir cnn_outputs/normalized/ --normalize_csv cnn_outputs/map_stats_downsampled.csv --labels labels.csv --weights_file weights_5e6.pth --batch_size 4 --outdir cnn_outputs
```

The quality of the predictions can be assessed using the diagnostic plots generated by eval_model.py. In order to generate these plots, we recommend that the user annotates the ground truth binarization thresholds for ~10-20 maps. We provide a template.csv that users should use to supply their annotated labels in the correct format to ```siren eval_model```.

In case the predictions are not accurate enough, the user may wish to try annotating the ground truth binarization thresholds for ~20-40 maps and fine-tuning the pre-trained model on this selected number of maps. Note that the fine-tuning functionality has not been extensively tested.

```  
fine_tune --help

optional arguments:
  -h, --help            show this help message and exit
  -voldir VOLDIR        Path to subset of downsampled and normalized input volumes
  -labels_csv LABELS_CSV
                        Path to .csv containing subset of normalized labels
  -batch_size BATCH_SIZE
                        Minibatch size
  -num_epochs NUM_EPOCHS
                        Number of epochs
  -weights WEIGHTS      Path to model weights
  -lr LR                Learning rate for fine-tuning
  -outdir OUTDIR        Path to output directory
```  
  
e.g.
  
```

siren fine_tune --voldir cnn_outputs --labels_csv normalized_labels.csv --num_epochs 5 --weights weights_5e6.pth --outdir fine_tuned_cnn

```

```siren eval_model``` can likewise be used to evaluate the quality of the fine-tuned predictions.


**3) Select a random subset of the occupied voxels, identify statistically significant co-occupancy relationships between these voxels, and cluster them to produce initial blocks** 
 
```
siren sketch_communities --help

optional arguments:
  -h, --help            show this help message and exit
  --voldir VOLDIR       Directory where (downsampled) volumes are stored
  --threads THREADS     Number of threads for multiprocessing
  --apix APIX           Angstroms per pixel of maps
  --bin BIN             Threshold at which to binarize maps
  --outdir OUTDIR       Directory where outputs will be stored
  --posp POSP           P value threshold before Bonferroni correction for positive co-occupancy
  --negp NEGP           P value threshold before Bonferroni correction for negative co-occupancy
  --binfile BINFILE     csv file containing predicted binarization thresholds
  --filter              If predicted binarization file is supplied, this flage indicates that volumes should be filtered by avg +/- 2 std of predicted threshold
  --subsample SUBSAMPLE  divisor for subsampling voxels
```  
e.g.
  
```
siren sketch_communities --voldir reconstruct_000000 --apix XX --threads 20 --binfile cnn_outputs/predictions.csv --outdir 01_siren 

```  

The ```--posp```, ```---posp_factor```, ```--negp```, and ```--negp_factor``` flags are tunable parameters; while we have set default values that typically work well, users may wish to screen a range of these values to find the optimal set of values for their data. In particular, loosening some of these parameters might be beneficial in cases of extensive conformational heterogeneity.


**4) Query every voxel against each initial block to produce expanded blocks** 
  
```
siren expand_communities --help

optional arguments:
  -h, --help           show this help message and exit
  --config CONFIG      Path to sketch_communities.py config file
  --blockdir BLOCKDIR  Path to directory where segmented blocks are stored
  --threads THREADS    Number of threads for multiprocessing
  --exp_frac EXP_FRAC
  --posp POSP          P value threshold before Bonferroni correction for positive co-occupancy
  --negp NEGP          P value threshold before Bonferroni correction for negative co-occupancy
  --filter             If predicted binarization file is supplied, this flag indicates that volumes should be filtered by avg +/- 2 std of predicted threshold
```  
 e.g.   
   
 ```
siren expand_communities --config 01_siren/00_sketch/config.pkl --blockdir 01_siren/00_sketch --threads 20

 ```  

As before, the ```--posp```, ```--posp_factor```, ```--negp```, and ```--negp_factor``` parameters are tunable, as is the ```exp_frac``` value. 

## Analysis of results
Users are recommended to first view the structural blocks directly in ChimeraX, and re-run if necessary with updated parameters. When a suitable set of parameters has been identified, the resulting structural blocks can be supplied to [MAVEn](https://www.github.com/lkinman/MAVEn) to be used as masks in occupancy calculations. Doing so will allow users to measure block occupancies across the ensemble, identify relationships between occupancies of different blocks, and identify subsets of the original volume ensemble with high occupancy of specific blocks of interest.
