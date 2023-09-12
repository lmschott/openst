# Open-ST: Computational Protocol

## Introduction

(explain the aim)

(explain the general steps)

1. Preprocessing of spatial transcriptomics data
2. Preprocessing of imaging data
3. Alignment of imaging and spatial transcriptomics
4. Assigning transcripts to segmented cells
5. (Optional) 3D reconstruction from serial sections of spatial transcriptomics and H&E images

## Requirements
The processing of open-ST data relies on several computational steps that can be installed using *conda* or *mamba* (our recommendation). Make sure that this is installed on your machine, following the adequate [instructions] for your operating system. 

Once you have installed *conda* or *mamba*, create an environment using the provided [environment.yaml] as a template. Download this file, browse to the folder where it was downloaded, and create the environment with the following command.

```bash
conda ... (from yaml)
```

With this, the openst package has been installed into a new conda environment called openst. Before proceeding, make sure the environment is active.

```bash
conda activate openst
```

Then, there are a few more dependencies that need to be installed or downloaded. First of all, spacemake:

```bash
conda ... spacemake
```

If you run into issues, refer to the [official spacemake documentation]

### Optional dependencies
There are two dependencies that need to be installed, for image stitching (based on ImageJ):

```bash
# download the proper for the platform
```

And, in the case of performing 3D registration of consecutive serial sections, STIM (Preibisch, Karaiskos, Rajewsky) needs to be installed. Follow the instructions provided on the [official github repository].

## 1. Preprocessing sequencing data
(we use spacemake from the fastq reads)

### 1.1 Running spacemake
You can find more instructions on the spacemake documentation. Here we provide an example, refer there for the complete documentation.

### 1.2 Quality control
We check the sample quality (e.g., PCR bias), and the stitching files

## 2. Preprocessing of imaging data
We'll start with the initial processing of H&E-stained tissue sections, as a prerequisite to aligning spatial transcriptomics data with tissue staining images.

### 2.1 Stitching (microscope-dependent)
To begin, stitch together the tile-scan images of your H&E-stained tissue sections using the Grid/Collection stitching plugin included in Fiji 1.53t. This will create a composite image of the entire section.

*Warning: this step depends on the microscope used for imaging. In our protocol, we use the (XXX microscope), and provide code to perform the stitching on any computer. Refer to the documentation or vendor of your microscope for the stitching of tile-scans*

```bash
openst image_stitch --microscope='keyence' --imagej-bin=<path_to_fiji_or_imagej> --tiles-dir=<path_to_tiles> --tiles-prefix=<to_read> --tmp-dir=<tmp_dir>
```

### 2.2 (Optional) Style transfer
Optionally, you can perform style transfer on your tile-scan images using a custom Contrastive Unpaired Translation (CUT) model. This can help equalize the style between sections and remove artifacts.

```bash
openst image_preprocess --input=<path_to_input_image> --CUT --CUT-model=<path_to_model> --output=<path_to_output>
```

### 2.3 Segmentation of staining image
Next, segment the nuclei in your images using Cellpose 2.2. We provide a model optimized for segmentation of fresh-frozen, H&E-stained tissue. You can specify any other model that works best for your data; refer to the cellpose documentation.

```bash
openst segment --input=<path_to_input_image> --model=<path_to_model_or_name> --output=<path_to_output> --dilate=<how_much_to_dilate_the_mask> --diameter=20 --mask-tissue --metadata=<where_to_write_metadata>
```

#### Extra: segmentation of very large cells
If your samples contain very large cells that cannot be segmented with the provided H&E model (e.g., adipocytes), you can perform a second round of segmentation with a cellpose model, adjusting the diameter parameter.

```bash
openst segment --input=<path_to_input_image> --model=<path_to_model_or_name> --output=<path_to_output> --dilate=<how_much_to_dilate_the_mask> --diameter=50 --metadata=<where_to_write_metadata>
```

Then, you can combine the segmentation masks of both diameter configurations.

```bash
openst segment_merge --inputs=<path_to_input_images_as_list_more_than_one> --metadata=<where_to_write_metadata>
```

Finally, you can create a report from the segmentation results

```bash
openst report --metadata <path_to_metadata> --output=<path_to_html_file>
```

This command will apply an "AND" between all images, to only preserve mask of non-overlapping, with the hierarchy provided in the files

## 3. Alignment of imaging and spatial transcriptomics
In order to assign transcripts to the nuclei segmented from the staining images, the pairwise alignment between the imaging and spatial transcriptomics modality must be performed. For this, we provide software that allows to:
1. Creation of pseudoimages from the spatial transcriptomics data.
2. Two-step (coarse, then fine) alignment of H&E images to pseudoimages of ST data.
3. Manual curation and refinement of the alignment for more precision (~0.5 µm error), by leveraging fiducial markers visible from both modalities.

### 3.1 Pairwise alignment
```bash
openst pairwise_aligner --args --metadata=<where_to_write_metadata>
```
#### 3.2 Quality Control
```bash
openst report --metadata=<where_to_write_metadata> --output=<path_to_html_file>
```
#### 3.3 (Optional) Refinement of alignment
Use the provided notebook that spawns a napari environment for visual assessment and manual refinement of the alignment, if necessary.
```bash
openst alignment_refiner --args --metadata=<where_to_write_metadata>
```

We can create the report again, to finally validate the refinement as we did for the pairwise alignment
```bash
openst report --metadata=<where_to_write_metadata> --output=<path_to_html_file>
```

## 4. Assigning transcripts to segmented cells
Finally, create spatial cell-by-gene expression matrices by aggregating the initial NxG matrix into an MxG matrix, where N maps to M via the segmentation mask. This step allows you to associate capture spots with segmented cells.
```bash
openst transcript_assign --args --metadata=<where_to_write_metadata>
```

We can create a HTML report which contains basic metrics for the file after transcript assignment
```bash
openst report --metadata=<where_to_write_metadata> --output=<path_to_html_file>
```

That concludes the preprocessing of imaging data in the Open-ST computational protocol. 

## 5. (Optional) 3D reconstruction from serial sections of spatial transcriptomics and H&E images
In this section, we will guide you through the process of creating a 3D reconstruction from serial sections of spatial transcriptomics (ST) and H&E images using the Spatial Transcriptomics ImgLib2/Imaging Project (STIM, v0.2.0). This reconstruction allows you to gain a comprehensive understanding of your biological samples in three dimensions.

### 5.1 Creation of csv files
Use the provided script

```bash
openst to_3d_registration --args --metadata=<where_to_write_metadata>
```

### 5.2 Conversion to n5 format
Convert the coordinate and gene expression information of these datasets into the n5 format, which is optimized for efficient image processing, using the st-resave function.
```bash
STIMBINS="/home/dleonpe/data/bin"
STIMINFILES="/data/rajewsky/home/dleonpe/projects/openst_paper/data/2_downstream/fc_sts_63/aligned_sections/1_input"
STIMOUTFILES="/data/rajewsky/home/dleonpe/projects/openst_paper/data/2_downstream/fc_sts_63/aligned_sections/2_stim_dataset"
FNAME="stitched_spots_merged_aligned_10px_GAN_segmented.h5ad."

$STIMBINS/st-resave \
    -i "$STIMINFILES/fc_sts_63_2_${FNAME}locations.csv,$STIMINFILES/fc_sts_63_2_${FNAME}genes.csv,fc_sts_63_02" \
    -i "$STIMINFILES/fc_sts_63_3_${FNAME}locations.csv,$STIMINFILES/fc_sts_63_3_${FNAME}genes.csv,fc_sts_63_03" \
    -o "$STIMOUTFILES/fc_sts_63.n5" \
    --normalize
```

### 5.3 Pairwise alignment
Utilize the st-align-pairs function to perform pairwise alignment of three sections below and above each section (r=3). This function creates image channels of gene expression for prespecified genes, aggregated per cell as a Gauss rendering around centroids, parametrized with a smoothness factor.

```bash
STIMBINS="/home/dleonpe/data/bin"
STIMOUTFILES="/data/rajewsky/home/dleonpe/projects/openst_paper/data/2_downstream/fc_sts_63/aligned_sections/2_stim_dataset"

# 2. Run pairwise alignment
$STIMBINS/st-align-pairs \
     -i "$STIMOUTFILES/fc_sts_63.n5" \
     --scale 0.03 \
     -r 3 \
     --hidePairwiseRendering \
     --overwrite \
     -sf 4.0 \
     --minNumInliers 15 \
     --numGenes 0 \
     -g 'KRT6A,KRT6B,S100A2,LYZ,CD74,IGKC,IGHG1,IGHA1,JCHAIN,CD74,AMTN'
     #--numGenes 20 \

```

### 5.4 Feature Filtering and Global Alignment
Filter the resulting set of feature matches between pairs of sections using an affine model. Configure the st-align-pairs function with appropriate parameters, such as --minNumInliers 15, --scale 0.03, and -sf 4.0 (smoothness factor).
```bash
STIMBINS="/home/dleonpe/data/bin"
STIMOUTFILES="/data/rajewsky/home/dleonpe/projects/openst_paper/data/2_downstream/fc_sts_63/aligned_sections/2_stim_dataset"

$STIMBINS/st-align-global \
     -i "$STIMOUTFILES/fc_sts_63.n5" \
     --skipICP \
     -g 'KRT6A,KRT6B,S100A2,LYZ,CD74,IGHG1,IGHA1,JCHAIN,CD74,AMTN'
```

### 5.5 Conversion to h5ad Format
Convert the n5 container back to the h5ad format for subsequent downstream analyses. This will also transfer the transformation models from the ST alignment onto the preprocessed and background-removed H&E images. This script will output the aligned spatial coordinates and an image volume that can be used for subsequent 3D visualization, of spatial transcriptomics and H&E staining in a common coordinate system.
```bash
openst from_3d_registration --args --metadata=<where_to_write_metadata>
```

You can quickly generate a HTML report to visualize the alignment quality. This will provide, for instance, a volumetric rendering on your browser using the channels (genes) selected for registration. This will also visualize the sections individually, to assess the deformations per section after registration, as well as the deformation fields.

```bash
openst report --metadata=<where_to_write_metadata> --output=<path_to_html_file>
```

With these steps completed, you will have successfully reconstructed a 3D representation of your biological samples, integrating spatial transcriptomics data and H&E images. This 3D reconstruction provides valuable insights into the spatial distribution of gene expression within your samples and enhances your understanding of complex biological structures.

### 5.7 3D visualization with ParaView
(explain)

## FAQ