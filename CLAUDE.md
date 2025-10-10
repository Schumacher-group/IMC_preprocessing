# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

IMC_preprocessing is a pipeline for Imaging Mass Cytometry (IMC) data processing. It extracts and preprocesses IMC data from MCD files through multiple stages: extraction, denoising with IMC Denoise, optional post-processing (CLAHE/normalization), cell segmentation with Mesmer, and cell table generation using ark-analysis.

## Multi-Environment Architecture

**Critical**: This project requires 3 separate conda environments due to incompatible dependencies:

1. **IMC_Denoise** (Python 3.9): For denoising operations
2. **deepcell** (Python 3.9): For Mesmer segmentation
3. **ark_analysis**: For cell table generation

The package must be installed in each environment separately with `pip install -e .`

## Common Commands

### Main Pipeline
```bash
# Run full pipeline (edit scripts/configs/config.yaml first)
python scripts/main.py

# Run with custom config
python scripts/main.py --config path/to/config.yaml
```

### Individual Steps
```bash
# Cell segmentation (requires deepcell environment)
conda activate deepcell
python scripts/Mesmer.py [--config configs/config.yaml]

# Cell table generation (requires ark_analysis environment)
conda activate ark_analysis
python scripts/cell_table.py [--config configs/config.yaml]
```

### Standalone Extraction
```bash
# Can be run independently if needed
python src/imc_preprocessing/extraction.py --config configs/config.yaml
```

## Configuration System

All pipeline parameters are controlled via `scripts/configs/config.yaml`:

- **root_data_folder**: Path to folder containing IMC_data directory
- **extraction**: Options: 'no', 'all', 'mcd_2_ome_tiff', 'ome_tiff_2_tiff', 'rename_leap_id'
- **IMC_Denoise.skip**: Set to True to bypass denoising
- **IMC_Denoise.retrain**: Force retrain models (ignores existing weights)
- **IMC_Denoise.channels_to_exclude**: Comma-separated channel names (e.g., 'Carboplatin')
- **Processing.skip**: Set to True to bypass post-processing
- **Processing.mode**: 'CLAHE' or 'unit' for normalization
- **Mesmer.skip**: Set to True to bypass segmentation
- **Mesmer.gpu**: Set to False to force CPU mode (set `CUDA_VISIBLE_DEVICES=-1`)

Logging output is written to `scripts/logging.log`.

## Pipeline Architecture

### Entry Point Flow
`scripts/main.py` orchestrates the pipeline in this order:
1. `process_response_metadata()` - Creates biosamples metadata file if missing
2. `extract_mcd_to_tiff()` - Calls `imc_preprocessing.extraction.main()`
3. `IMC_Denoise_transformation()` - Calls `Denoise_train.main_train()` then `Denoise_predict.main()`
4. `post_denoise_transformation()` - Calls `post_Denoise_processing.main()`

Note: Mesmer and cell_table are run separately via their own scripts.

### Core Modules

**src/imc_preprocessing/extraction.py**
- `mcd_2_ome_tiff()`: Extracts OME-TIFF from MCD files using imcsegpipe wrapper
- `ome_tiff_2_tiff()`: Converts OME-TIFF to split/combined TIFF formats with Leap ID correction
- `rename_leap_id()`: Applies dataset-specific file renaming/deletion rules
- Contains extensive hardcoded logic for correcting sample naming issues (Leap003/004 swaps, Leap091/092, etc.)

**src/imc_preprocessing/Denoise_train.py**
- `process_channel()`: Per-channel training pipeline - generates patches, trains DeepSNiF model, saves weights
- `main_train()`: Iterates over all channels, skips already-trained channels unless `retrain=True`
- Training weights saved to `Save_directory/weights_folder/weights_{channel_name}.keras`

**src/imc_preprocessing/Denoise_predict.py**
- Applies trained models to denoise images
- Outputs to `output_directory` configured in yaml

**src/imc_preprocessing/post_Denoise_processing.py**
- Applies contrast adjustment (CLAHE) or unit normalization
- Processes entire directory of denoised images

**scripts/Mesmer.py**
- `create_image_file_record()`: Scans for membrane/nuclear marker images
- `load_imgs_and_concatenate()`: Combines multiple marker channels with fuzzy logic averaging
- `segment()`: Runs deepcell.applications.Mesmer on combined nuclear+membrane images
- Hardcoded markers: membrane (CD14, CD11b, CD45, etc.), nuclear (DNA1, DNA2)
- Outputs: segmentation masks to `deepcell_out_path`, combined channel images to `mask_path`

**scripts/cell_table.py**
- Wrapper around `ark.segmentation.marker_quantification.generate_cell_table()`
- Generates size-normalized and arcsinh-transformed cell tables
- Requires all FOVs to have identical channel sets

**src/imc_preprocessing/imcsegpipe/**
- Wrapper around steinbock's MCD extraction functionality
- `extract_mcd_file()`: Core function called by extraction.py

## Directory Structure Expectations

```
root_data_folder/
├── IMC_data/
│   └── Leap001/
│       └── Leap001.mcd
├── split_channels_nohpf/     # Split TIFF output
│   └── Leap001_1/
│       ├── CD14.tiff
│       └── ...
├── combined_tiff/             # Multichannel TIFF output
│   └── Leap001/
│       └── Leap001_1.tiff
└── Img_Denoised/              # Denoised output
```

Acquisitions from a single MCD file are saved with format `LeapXXX_Y` where Y is acquisition ID.

## Key Implementation Details

- **Memory management**: Explicit `gc.collect()` and `tf.keras.backend.clear_session()` calls throughout to prevent OOM
- **GPU control**: Set `CUDA_VISIBLE_DEVICES=-1` in Mesmer script when `gpu: False` in config
- **Image filtering**: Images < 128 pixels per side are skipped during extraction
- **Marker filtering**: Channels with 'LASER' or 'TEST' in description are excluded
- **Mesmer inputs**: Stacks nuclear markers (DNA1, DNA2) and membrane markers into 2-channel input
- **IMC Denoise**: Trains separate model per channel, saves weights as `weights_{channel}.keras`
- **Batch processing**: DeepSNiF training uses batch size from config, prediction also batched

## Testing

Tests are located in `tests/test.py` but appear minimal. No comprehensive test suite exists.

## Dependencies

Key external libraries:
- tensorflow >= 2.17.0 (with keras >= 3)
- deepcell == 12.6 (separate environment)
- IMC_Denoise (external package, see their repo)
- ark-analysis (for cell tables, separate environment)
- steinbock (wrapped via imcsegpipe)
- readimc, tifffile, xtiff for image I/O
