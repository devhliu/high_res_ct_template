---
title: A Publicly Available, High Resolution, Unbiased CT Brain Template
author:
  - name: John Muschelli
    email: jmusche1@jhu.edu
    affiliation: JHSPH
address:
  - code: JHSPH
    address: Johns Hopkins Bloomberg School of Public Health, Department of Biostatistics, 615 N Wolfe St, Baltimore, MD, 21205
abstract: |
  Clinical imaging relies heavily on X-ray computed tomography (CT) scans for diagnosis and prognosis.  Many research applications aim to perform population-level analyses, which require images to be put in the same space, usually defined by a population average, also known as a template.  We present an open-source, publicly available, high-resolution CT template. With this template, we provide voxel-wise standard deviation and median images, a basic segmentation of the cerebrospinal fluid spaces, including the ventricles, and a coarse whole brain labeling. This template can be used for spatial normalization of CT scans and research applications, including deep learning. The template was created using an anatomically-unbiased template creation procedure, but is still limited by the population it was derived from, an open CT data set without demographic information. The template and derived images are available at https://github.com/muschellij2/high_res_ct_template. 
journal: "International Conference on Information Processing and Management of Uncertainty in Knowledge-Based Systems"
date: "2020-02-18"
bibliography: refs.bib
output: 
  bookdown::pdf_book:
    base_format: rticles::elsevier_article
    keep_tex: true
    number_sections: yes
    keep_md: true
  html_document: 
    toc: true
---



# Introduction

Many research applications of neuroimaging use magnetic resonance imaging (MRI).  MRI allows researchers to study a multitude of applications and diseases, including studying healthy volunteers as it poses minimal risk.  Clinical imaging, however, relies heavily on X-ray computed tomography (CT) scans for diagnosis and prognosis.  Studies using CT scans cannot generally recruit healthy volunteers or large non-clinical populations due to the radiation exposure and lack of substantial benefit.  As such, much of head CT data is gathered from prospective clinical trials or retrospective studies based on health medical record data and hospital PACS (picture archiving and communication system).  Most of this research is on patients with neuropathology, which can cause deformations of the brain, such as mass effects, lesions, stroke, or tumors.

Many clinical protocols perform axial scanning with a high within-plane resolution (e.g.$0.5$mm x $0.5$mm) but lower out-of-plane resolution (e.g. $5$mm).  High resolution scans (out of plane resolution $\approx 0.5$mm) may not be collected or reconstructed as the lower resolution scans are typically those read by the clinician or radiologist for diagnosis and prognosis. Recently, a resource of a large number of CT scans were made available, denoted as CQ500 [@cq500].  These scans include people with a number of pathologies, including hemorrhagic stroke and midline shifts. Fortunately, this data also includes people **without indicated pathology** with high resolution scanning, which is what we will use in this study.  

The goal of this work is to create an anatomically unbiased, high-resolution CT template of the brain.  That is, we wish to create a template that represents the population, regardless of any initial templates we start with.  The first, and we believe the only, publicly-available CT template was released by @rorden_age-specific_2012 (https://www.nitrc.org/projects/clinicaltbx/).  That template was created with the specific purpose of creating a template with a similar age range as those with stroke, using 30 individuals with a mean age of 65 years old (17 men).  The associated toolbox released contained a high resolution (1x1x1mm) template, with the skull on, in Montreal Neurological Institute (MNI) space.  Subsequent releases have included skull-stripped brain templates, but only in a lower (2x2x2mm) space (https://github.com/neurolabusc/Clinical).  This lower resolution template matches what is used in many MRI and functional MRI analyses.

Thus, the current CT templates available are a high-resolution template (1mm$^3$), but not of the brain only (and skull stripping the template performs marginally well), and a low-resolution template of the brain only, both in MNI space.  We have used these templates in previous analyses, but would like a brain template that was 1) constructed using an unbiased anatomical procedure, 2) uses more patients, 3) uses high-resolution scans to achieve a higher resolution, and 4) provide an image which dimensions are easily used in deep learning frameworks.

As the CQ500 data was released under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0 (CC-NC-SA) International License, we can release the template under the same license.

## Methods

All code, analysis, and reporting was done the R statistical programming language [@RCORE] and a number of packages from the R medical imaging package platform Neuroconductor [@neuroconductor].  



### Data

We defined a high-resolution patient scan as having a within-axial resolution of $0.7$x$0.7$mm or less, with full coverage of the brain.  For example, if the cerebellum was not imaged, that image was discarded.  All scans were non-contrast CT scans with a soft-tissue convolution kernel.  As CT scans are generally well calibrated across sites and are measured in standard units of Hounsfield Units (HU), no intensity normalization was done.  Intensities less than $-1024$ HU (the value for air) and greater than $3071$ HU were Winsorized to those values, as values outside of these are likely artifact or areas outside the field of view.  

All data was converted from DICOM files to NIfTI (Neuroimaging Informatics Technology Initiative) using `dcm2niix` [@dcm2niix] using the `dcm2niir` package [@dcm2niir].  This conversion corrects for any gantry tilt and enforces one fixed voxel size for the image, which is necessary if different areas of the image are provided at different resolutions, which is sparsely seen in clinical CT images.

From the CQ500 data set, 222 subjects had no indication of pathology, of which 141 had a high-resolution scan (if multiple were present, the one with the highest resolution was used).  From these 141 people, 130 had "thick-slice" scans where the out-of-plane resolution was greater than $4$mm.  We used these 130 scans for construction of the template.  The 11 scans were discarded as we wish to perform the same operation using low-resolutions scans to see the effect of initial resolution on template creation, but that is not the focus of this work.

For all images, the head was skull-stripped so that only brain tissue and cerebrospinal fluid (CSF) spaces were kept, using a previously validated method [@muschelli_validated_2015] using the brain extraction tool (BET) from FSL (FMRIB Software Library
) [@smith_fast_2002; @jenkinson_fsl_2012].  We chose an image (patient 100 from CQ500), for template creation.  This choice was based on a within-plane resolution close to $0.5$x$0.5$mm ($0.488$x$0.488$mm), an axial slice size of 512x512, and an out-of-plane resolution of $0.5$mm.  The image was resampled to $0.5$x$0.5$x$0.5$mm resolution so that the voxels are isotropic.  We would like the image to be square; we padded the image back to 512x512 after resampling, and the image had 336 coronal-plane slices. 

### Template Creation

The process of template creation can be thought of as a gradient descent algorithm to estimate the true template image as inspired by the advanced normalization tools (ANTs) software and the R package ANTsR that implements the registration and transformation was used (https://github.com/ANTsX/ANTsR) [@avants_reproducible_2011].  The process is as follows:

1. Let $I_{i}$ represent the image where $i$ represents subjects.  We registered all images to the template, denoted $\bar{T}_{k}$ where $k$ represents iteration, using an affine registration followed by symmetric normalization (SyN), a non-linear deformation/diffeomorphism, where the composed transformation is denoted as $G_{i, k}$ [@avants_symmetric_2008].  Let the transformed image be denoted as $T_{i, k}$.  In other words, $I_{i}\overset{G_{i,k}}{\rightarrow}T_{i, k}$.  The transformation $G_{i, k}$ is represented by a 4D warping image.  Let $T_{1}$ be the original template chosen above and $G_{i, 1}$ be the transformation for an image to the original template.
2.  Calculate a the mean, median, and standard deviation images, where the mean image is $\bar{T}_{k} = \frac{1}{n} \sum\limits_{i = 1}^n T_{i, k}$, using a voxel-wise average.  
3.  Calculate the average warping transformation: $\bar{G}_{k} = \frac{1}{n} \sum\limits_{i = 1}^n G_{i, k}$.  A gradient descent step size of 0.2 was specified for SyN gradient descent, such that:
$\bar{T}_{k + 1} = \bar{T}_{k} \times \left(-0.2 * \bar{G}_{k}\right)$.  The median and standard deviation are transformed accordingly.

For each iteration $k$, we can calculate a number of measures to determine if the template has converged compared to the previous iteration $k - 1$.  We calculated the Dice Similarity Coefficient (DSC) between the mask of iteration $k$ and $k-1$, where the mask for iteration $k$ is defined as $\bar{T}_{k} > 0$.  The DSC measures if the overall shape is consistent across iterations.  We also the root mean squared error (RMSE) of voxel intensities, e.g. $\frac{1}{V} \sum\left(\bar{T}_{k} - \bar{T}_{k-1}\right)^2$, where $V$ is the number of voxels in the volume.  The RMSE can be calculated over a series of volumes, either 1) the entire image, 2) over the non-zero voxels in iteration $k$, 3) in iteration $k-1$, or 4) the union (or intersection) of the 2 masks.  Calculation over the entire image gives an optimistic estimate as most of the image are zeroes, and the choice of either iteration $k$ or $k-1$ masks is arbitrary, so we calculated the RMSE over the union of the 2 masks.  The RMSE represents if the values of the image are close across iterations.



To define convergence, we would like a high DSC between the masks and a low RMSE.  Ideally, the convergence criteria would set a DSC of $1$ and a RMSE less than $1$ Hounsfield Unit (HU), which would indicate the voxel intensity is changing less than $1$ HU on average.  As CT scans are measured in integers, this RMSE would likely be as good as possible.  We set a DSC cutoff of $0.95$ and chose the template with the lowest RMSE.  As this procedure is computationally expensive, we ran $40$ iterations, which was adequate for achieving stable results (Figure \@ref(fig:performance)).  

Values of the final template that were lower than $5$ HU were boundary regions, outside the region of the brain and likely due to average of one or a small few of images, incongruent with the remainder of the template (Supplemental Figure 1).  We did not constrain the DSC and RMSE calculation excluding these regions, but excluded values less than $5$ HU from the final template.

After the template was created, we padded the coronal plane so that the template was 512x512x512.  The intention is that these dimensions allow it easier to create sub-sampled arrays that are cubes and multiples of 8, such as 256x256x256, 128x128x128, or 64x64x64 with isotropic resolution.  

### Segmentation

Though the template itself is the main goal of the work, many times researchers use or are interested in annotations/segmentations of the template space.  The contrast between gray matter and white matter in CT imaging is not as high as T1-weighted MRI.  Some areas, such as the cerebellum, corpus callosum, and basal ganglia can be delineated well.  Thus, segmentation methods based on intensity may not differentiate gray and white matter adequately.  We instead used a multi-atlas registration approach using previously-published set of 35 MRI atlases from @bennett2012miccai, which had whole brain segmentations, including tissue-class segmentations.  

We registered each brain MRI to the CT template using SyN and applied the transformation to the associated tissue segmentation and whole brain segmentation from that MRI template.  Thus, we had 35 tissue segmentations of the CT template in template space, and the segmentations were combined using STAPLE [@warfield2004simultaneous] via the `stapler` package [@stapler].  The whole brain structures were combined using majority vote. 

Separating the brain from the cerebrospinal fluid areas (mainly ventricles) are of interest in many applications, such as Alzheimer's disease [@de1989alzheimer; braak1999neuropathology].  In addition, we segmented the template using Atropos [@atropos], which used a k-means clustering approach with 2 clusters (CSF/tissue) to obtain a CSF mask.  Additionally, we registered the MNI T1-weighted template to the CT Template using SyN, and applied the transofrmation used the ALVIN (Automatic Lateral Ventricle delIneatioN) mask of the ventricles [@alvin].  We masked the CSF mask with this transformed ALVIN mask to get a mask of lateral ventricles as well. 


## Results



As we see in Figure \@ref(fig:performance)A, the DSC quickly increases and reaches a high score, where the horizontal line indicates a DSC of $0.99$.  The red dot and vertical line indicate the iteration that had the maximum DSC (0.9896).  As the DSC is high for all iterations past iteration $15$, we chose the template based on the minimum RSE.  In Figure \@ref(fig:performance)B, we see a similar pattern of improving performance, but by lowering the RMSE.  The lowest RMSE is noted by the red point with a value of $1.47$.  Thus, this iteration (iteration $37$) is the template we will choose.




![(\#fig:performance)Convergence of Shape and Intensity of the Template over Iterations.  Here we see the Dice Similarity Coefficient (DSC) increase between an iteration and the previous iteration, achieving high degrees of overlap, indicating the shape of the surface of the image is similar and converging (panel A).  We also see the root mean-squared error (pane) drops as the iterations increase and then levels off around 4 Hounsfield units (HU), the horizontal line.  The red dot indicates the iteration chosen for the template.](index_files/figure-latex/performance-1.pdf) 





![(\#fig:template)Template Image, Standard Deviation Image, and Histogram of Intensities.  Here we show the template in the left panel, the voxel-wise standard deviation, denoting areas of variability (which include biological and technical variability), and the histogram of the template intensities/Hounsfield Units (HU).  Overall the template is smooth and values fall in the range of 5 to 65 HU.](index_files/figure-latex/template-1.pdf) 


The template for this image can be seen in Figure \@ref(fig:template), along with the standard deviation image, and a histogram of the intensities of the template.  Areas outside the brain mask were removed for visualization.  We see the template is relatively smooth, with values from 5 HU to around 65 HU.  The standard deviation image shows high variability around the lateral horns, which may be due to calcifications in a set of patients, which have abnormally high HU values.  The high standard deviation areas near the midline are likely due to dense areas of the falx cerebri, including potential falx calcifications.


![(\#fig:seg)Template Image, Tissue Segmentation, Whole Brain Segmentation, and Lateral Ventricle Segmentation.  We see the areas of white matter, gray matter, cebebrospinal fluid (CSF) in Panel B. We see the whole brain structural segmentation in Panel C, and the lateral ventricle segmentation from Atropos in Panel D. ](index_files/figure-latex/all_segs.pdf) 

In Figure\@ref(fig:seg), we see the template again, with the tissue-class segmentation (Panel B), whole brain structural segmentation (Panel C), and Atropos lateral ventricle segmentation.  Overall, we see some differences between the segmentation of the CSF based on Atropos and the multi-atlas labeling approach.  We have provided a lookup table for each structure label with its corresponding value in the image. 


## Discussion 

We present a high-resolution, publicly-available CT template with associated segmentations and other annotations of the template. The data used was from a publicly-available dataset, the CQ500.  The main downside with the CQ500 data set is that no demographic or clinical information was released for each patient, save for indication for pathology.  Therefore, we cannot attest the general population of interest for this template.  Furthermore, we cannot fully assume these patients were disease-free as a lack of pathology only applies to the categories of interest in the CQ500 dataset (intracranial/subdural/subarachnoid/epidural hematoma, calvarial or other fractures, mass effect and midline shifts).   In future work, we hope to prepare age- and sex-specific templates for each population based on hospital scans and records, where we have demographic information and confirmation of lack of neuropathology. 


In addition to the template, we have provided a set of segmentations.  This includes a whole brain segmentation of over 150 structures. Though this may prove useful, we caution users to how well this template can provide an accurate segmentation of these structures.  At least, the accuracy of the segmentation may have variable accuracy at different areas of the brain.   

The resulting image dimensions was 512x512x512, with a resolution of 0.5x0.5x0.5mm.  The fact that the image dimension is a multiple of 8 allows it to be resampled to 1x1x1mm and 2x2x2mm and remain as a cube.  These dimensions are particularly important in certain deep learning architectures and frameworks.  Though most templates are given using the mean image, we believe the standard deviation image represents variability in the area.  This variability represents true systematic and biologic variability.  One important area of systemic variability is registration errors.  Therefore this template allows for the creation of z-score images, where a new image is registered to the mean image, the mean image is subtracted, and then divided by the standard-deviation image, so that voxels represent standard deviations away from the mean voxel.  This image may be a useful tool in feature extraction. Thus, we believe this template provides a standard, isotropic space that is conducive to machine learning and can reduce the burden of standardization for medical imaging applications.


CQ500 is Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License.  Therefore, the template is released under the same license.  The images are located on https://github.com/muschellij2/high_res_ct_template and can be accessed at https://johnmuschelli.com/high_res_ct_template/template/.


## Acknowledgments
This work has been been supported by the R01NS060910 and 5U01NS080824 grants from the National Institute of Neurological Disorders and Stroke at the National Institutes of Health (NINDS/NIH). 


References {#references .unnumbered}
==========

