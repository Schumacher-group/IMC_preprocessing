# IMC preprocessing
Extraction and preprocessing pipeline for IMC data. From a single mcd file, it extracts the tiff files using [steinbock](https://bodenmillergroup.github.io/steinbock/)  and run  [IMC denoise](https://github.com/PENGLU-WashU/IMC_Denoise) for denoising. To reduce batch effects, we also recommend applying contrast adjustment through Contrast Limited Adaptive Histogram Equalization.
You can also run cell segmentation using [Mesmer](https://github.com/vanvalenlab/deepcell-tf), and create a cell table for cell based analysis using [ark-analysis](https://github.com/angelolab/ark-analysis).

## Citation
This pipeline was used in the following preprint:

**"Identifying tissue states by spatial protein patterns related to chemotherapy response in triple-negative breast cancer"**
bioRxiv (2025). DOI: [10.1101/2025.10.06.680783](https://www.biorxiv.org/content/10.1101/2025.10.06.680783)

If you use this pipeline in your work, please consider citing the preprint. 

## Downstream analysis
Downstream analysis is handled in a different repository: https://github.com/Schumacher-group/IMC_TNBC_analysis
Prediction of therapy response based on the analysed data is handled in: https://github.com/Schumacher-group/ML4SpatialAnalysis

## Installation
If you want to run all of the steps above, there is not an environment that workas for all together. Therefore, I recommend you create 3 different python environments.
### Install dependences
1. Install [IMC Denoise](https://github.com/PENGLU-WashU/IMC_Denoise) ( I have used with python 3.9 and newer tf version then in their readme)
    ```
    conda create -n IMC_Denoise python=3.9
    #follow instructions from author's repo
    ``` 
1. Install [deepcell](https://github.com/vanvalenlab/deepcell-tf) from pip:
    ```
    conda create -n deepcell python=3.9
    conda activate deepcell
    pip install deepcell==12.6
    ```
1. Install ark analysis
### Install me
- Download the repo and install the package in  each and every environment.
```
$ git clone https://github.com/g-torr/IMC_preprocessing.git
$ cd IMC_preprocessing
$ conda activate IMC_Denoise 
$ pip install -e .
$ conda activate deepcell 
$ pip install -e .
$ conda activate #ark_analysis_env
$ pip install -e .

```
## How to use it
### Quickstart
First, edit the file `/scripts/configs/config.yaml`. This file set the parameters for the pipeline. Make sure that `root_data_folder` points to the folder that contains the mcd files. I recommend using absolute path, or you can use relative paths pointing from the `scripts/` folder. Use my configuration file as a guide
The general usage is: 
```
$ python scripts/main.py
```

## Directory structure of raw IMC images
The configuration file assumes that mcd data ar located in the folder `mcd_data_folder = IMC_data`, containing a structure like:
```
|---IMC_data
|---|---Leap001
|---|---|---Leap001.mcd
             ...
|---|---Leap002
|---|---|---Leap002_x.mcd
|---|---|---Leap002_y.mcd
|---IMC_preprocessing
|---|---|scripts|main.py
```
Each mcd file may contain several acquisitions. Acquisitions are saved in `tiff_folder_name_split` and `tiff_folder_name_combined` as grayscale and multichannel tiffs respectively. Here `a` and `b` represents acquisition ids.

```
|---IMC_data
|---IMC_preprocessing
|---tiff_folder_name_split
|---|---Leap001_a
|---|---|---|channel1.tiff
             ...
|---|---|---|channeln.tiff
|---|---Leap002_x_b
|---|---|---|channel1.tiff
             ...
|---|---|---|channeln.tiff

|---tiff_folder_name_combined
|---|---Leap001
|---|---|---Leap001_a.tiff
```
## Directory structure of images for  IMC denoise
IMC Denoise takes the images from `tiff_folder_name_split` folder, trains and predicts the new processed images to the `output_directory`. Parameters of IMC Denoise can be set up in the file `/scripts/configs/config.yaml`
Logging is saved in `/scripts/logging.log`

## Hot-pixel filtering
Hot-pixel removal in this pipeline is performed by the **DIMR** stage of [IMC_Denoise](https://github.com/PENGLU-WashU/IMC_Denoise) (Lu et al.), *not* by the steinbock spatial filter (`filter_hot_pixels` / `create_analysis_stacks` in `src/imc_preprocessing/imcsegpipe/`, which are defined but not called by the pipeline). The split-tiff folder name `split_channels_nohpf` is historical and refers only to the absence of the steinbock filter — DIMR is still applied on top, downstream.

The DIMR parameters used to produce the cell table are pinned in [`scripts/configs/config.yaml`](scripts/configs/config.yaml) under `IMC_Denoise.params`:

| Parameter      | Value | Meaning                                                                 |
| -------------- | ----- | ----------------------------------------------------------------------- |
| `n_neighbours` | 10    | Number of neighbouring pixels considered when flagging a hot pixel      |
| `n_iter`       | 3     | Number of DIMR iterations                                               |
| `window_size`  | 5     | Local window (pixels) over which the hot-pixel criterion is evaluated   |

The parameters were chosen by visual inspection together with a quantitative criterion: after filtering, the distribution of `(max − mean)` intensity inside 5×5 tiles should be continuous, i.e. no residual isolated bright outliers. DIMR is applied to every channel listed in the panel **except** those in `IMC_Denoise.channels_to_exclude` (Carboplatin, which is processed separately in the post-denoise step).

> **To reproduce the published cell table**, edit `scripts/configs/config.yaml` and set `IMC_Denoise.skip: False`. The default in the repo is `True` only so that downstream steps can be re-run without re-training DeepSNiF.

## Mesmer
## Mesmer
To run Mesmer, run:
```
conda activate deepcell
python scripts/Mesmer.py
```

