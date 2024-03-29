# Supplementary information to run stroke segmentation software
A repository for supplementary considerations required to run certain stroke segmentation algorithms.

## Run environment
- Debian 9.9 Stretch
- Xeon E5-1650 v4 3.60 GHz 12-Thread
- 64GB RAM
- R 3.3.3

# LINDA
The stroke segmentation software LINDA is already well documented (https://github.com/dorianps/LINDA) and easy enough to get running for native space segmentation, but could do with a few more comments regarding special user cases encountered.

The original paper is at: http://doi.org/10.1002/hbm.23110

A recent (anno 2019) comparison paper of LINDA with other stroke segmentation algorithms can be found at: http://doi.org/10.1002/hbm.24729

This paper found LINDA superior in most aspects, but not every aspect. Computational time was one issue, but with adequate computational power (medium end enterprise desktop anno 2019) it is not a significant issue compared to other algorithms in the neuroscience field, and considering the difficulty amassing many stroke cases for a study.

## Wrappers to make everything run smoother

Four steps of are certainly necessary: (Assuming BIDS)
1. Create new copies of original T1w images. Ensure a different, but consistent filename. (Python?; Easy to manipulate file systems with.)
2. Swapdir of each image. (BaSh?; The key is either way to use a shell with a path to FSL. Python and R has "system" commands, but not necessarily with FSL.)
3. Run the prediction. (R)
4. Read and amass relevant information. In this case number of voxels and another thresholding (Matlab?; Easy to read and do math on 3D+ arrays.)

A list should be made to denote which ID that has stroke on which hemisphere. This list can be implemented in either step 1 or 3.

Ideally everything should be handled in R for simplicity.

**PS! It has come to my attention that simply setting "cache = TRUE", presents an error in the "linda_predict" because it cannot find the output object "L" when the segmentation already exists (intentional from my part). This stops the whole LINDA wrapper for "no" reason, as the output path is of no interest when we want to use the probability prediction in native space. This can be bypassed by creating a new script with "R/aaa_utils.R/print_msg" [found in GitHub] function and "LINDA::linda_predict" [from inside R] or by commenting out the last "return(L)" line. Also remember to load the ANTsRCore, ANTsR and ITKR before running the ad-hoc (custom) hack of "linda_predict". See the issue-thread (https://github.com/dorianps/LINDA/issues/24) for potential development of the issue.**

## Libraries and additional software required 
There are two instances that may prove troublesome if you haven't already installed the R package "devtools" and "ANTsR"/"ITKR".

For "devtools" a few additional libraries may be required. Which libraries exactly will be stated in the error specification in the R console. This can be a tedious troubleshooting step depending on how many libraries you are missing. https://www.r-project.org/nosvn/pandoc/devtools.html is in agreement with the preceeding observation and will vary depending on the linux environment.

For ANTsR, ITKR and (or at least one of these?) LINDA you will require Cmake 3.10.2 or later. For Debian Stretch, you can only get 3.7.2 via the package manager. It is therefore required to download (https://cmake.org/download/) an adequate version yourself and place it under "/usr/local/cmake_3_X_X". Root access is necessary for this. Remember to export the "bin" folder inside "cmake_3_X_X" to your console and uninstall older versions to make it find this new version.

## Flipping images
To flip images we used FSL (https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/Fslutils), more specifically "fslorient". The author of FSL encourages that the user has read what the function "fslorient" actually do, because it can easily make unwanted changes to original data if you are not careful.

1. Make a copy of the original nifti image, such that you can keep the original image and for bilateral stroke segment (although bilateral stroke segmentation is not recommended).
2. Call "fslorient -swaporient copied_t1w.nii.gz".

A .sh script may be the most efficient to flip the required (or all) images.

## Speed, performance and core/thread usage comments
In the example run (done on a single core) at LINDA's GitHub page, the run took 3.5 hours. This may appear as "too much" in some situation. But, as experienced on our test rig, the computational time is at best 1/7th of this. It is unknown what Hz the CPU in the example run had.

### Test case 1 - Large stroke
Took 30.5 minutes. This ran without parallell specifications in R (lapply, etc), although many processes throughout the segmentation used all available threads. This may indicate that user specified parallell processing is not necessarily beneficial with a relatively small amount of cores (< 6 cores and < 12 threads).

The result appears fairly good for descriptive purposes when thresholded at 50% (or 0.5), but not close to voxel perfection. Prediction range was at 0 to 1.
Both the probability and program prediction was okay. Although it seems that the program prediction used something else than a 50% threshold, as it thought a larger area was stroke afflicted.

To clarify:
- probability prediction = Prediction3_probability_native
- program prediction = Prediction3_native

### Test case 2 - Medium stroke
Took 33.5 minutes. This ran without parallell specifications in R (lapply, etc), although many processes throughout the segmentation used all available threads. Both faster and slower than the small and large stroke cases, respectively.

Both the program prediction and the probability prediction was ok. 50% threshold was in this case ok, perhaps, due to a prediction range of 0 to 1. Still the program prediction denote a larger area again as afflicted by stroke.

There was a single false positive stroke cluster found. It seems like it mistook an "empty" dark cavity between the grey matter as stroke. This is weird, because its appearant opposite hemisphere also has such an "empty" cavity. Its size is negligiable for descriptive purposes considering the actual stroke site's cluster size.

### Test case 3 - Small stroke
Took 41.6 minutes. This ran without parallell specifications in R (lapply, etc), although many processes throughout the segmentation used all available threads. Definitely slower than the Large and Medium stroke cases.

It found the actual stroke site, but also found 3-4 other unrelated false positives. The one largest false positive  essensially doubled the true estimate was unfortunate, but not surprising that the algorithm thought at least parts of it was a stroke due to a local cortex cluster darkening on the T1w image. The remaining smaller errors were expected as with WMH segmentation algorithms, and were not of detrimental size.

A manually selected threshold of 5% (or 0.05) found the actual stroke site sufficiently, and the prediction range was 0 to 0.1783. This can pose a challenge for selecting a single threshold across stroke cases. 50% of max value in a prediction may be a decent threshold estimate.

It is reasonable to expect lower stroke estimates to be overestimates. If NULL cases (defined as range[prediction] = 0 to 0) are returned from the LINDA algorithm it is consequently also safe to say that it has underestimated the stroke.

Only the probability prediction was ok. The program prediction yielded a NULL case, which suggest that the built-in threshold is not necessarily optimal enough. 

This sub-optimailty in thresholding is especially true in situations where it is known apriori that all images supplied to the algorithm has confirmed ischemic strokes. The comparison paper, linked earlier, had an issue with quite a few NULL cases, some which potentially could have been avoided if a different thresholding scheme had been applied.

### Test case 4 - Bilateral medium stroke
Took 27.7 and 37.7 minutes to segment left and right hemispheres respectively. This ran without parallell specifications in R (lapply, etc), although many processes throughout the segmentation used all available threads.

The probability prediction found all apparent stroke sites. But, it did not manage to fill the whole extent of the strokes and had somewhat substantial wrongful segmentation. This is not unexpected, although ok to confirm.

It should be mentioned that the stroke sites were quite symmetric (2 of 4) between the hemispheres. As such, it is not strange that the prediction range suffered 0 to 0.916 [left] and 0 to 0.999 [right]. The idea with using a 50% of max range seems to suffer in this bilateral case, as one of the stroke sites vanished at 0.46 threshold [left]. Suggesting that a 50% relative threshold is too simple for the most accurate segmentation. On the contrary the [left] program prediction did find both stroke sites.

It is reasonable to assume that bilateral cases do underestimate the correctly detected stroke sites. It is perhaps also a question of defining what parts of an image could be defined as a stroke, and the algorithm may in itself not consider darker than penumbra areas to be a stroke area (dead tissue that is washed away/removed over time leaving a CSF cavity?).

## Storage consumption
Each new run/prediction is expected to use at least up to 200MB of storage. 3 out of 3 of our runs took this much space.
(Original nii.gz files use around 12MB for 176x256x256 INT16 images)
