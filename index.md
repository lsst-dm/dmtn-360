# The New MultiBandDetectionTask

```{abstract}
Currently detection on coadds is performed running `DetectCoaddSourcesTask`, which runs `DynamicDetectionTask` on each band and then `MergeDetectionsTask` to merge the catalogs into a single catalog. `MultiBandDetectionTask` is a single pipeline task that replaces those calls by running `MultiResolutionDetectionTask` on all bands simultaneously, creating a more complete and pure catalog. This technote describes these new tasks and offers a comparison with the legacy `DynamicDetectionTask` results. It also lists the shortcomings of the new detection task and how they will be addressed in the future.
```

## Introduction

### Why a new detection task

The Rubin Observatory LSST science pipelines currently have two different detection tasks that live in the `meas_algorithms` package: `SourceDetectionTask`, which is the base detection task that runs primarily on single visit images, and `DynamicDetectionTask`, which runs on coadds. `DynamicDetectionTask` traces its history back to the era when the science pipelines were being used to process data from the HyperSuprimeCam (HSC) camera on the Subaru telescope. Researchers on the HSC project noticed that many faint sources were not passing the detection cuts, resulting in several "hacks" being created in order to better detect sources when bright stars and galaxies are important. The TL;DR is that `DynamicDetectionTask` adds code to rescale the variance and subtracts a local background in order to create a more complete overall catalog. As LSST has begun to proces a much wider variety of fields it has become apparent that the hacks in `DynamicDetectionTask` are fragile at LSST scales, missing obvious detections in deep fields, creating multiple spurious detections in noisier fields, and missing a great number of detections in crowded fields.

### How the new tasks are different

Similar to the current scheme, there is a pipeline task, `MultiBandDetectionTask`, that calls a descendent of `SourceDetectionTask` called `MultiResolutionDetectionTask`. As described in (reference here), `MultiResolutionTask` uses starlets, a particular undecimated wavelet that has been proven useful in processing astrophysical images (cite starck papers here), which in the case of detection work as compensated filters. This allows them to essentially do a local background subtraction for free and results in detection catalogs that are more complete with fewer spurious sources (therefore higher purity).

## The algorithms involved

### Detection with starlets

Detection uses starlets created using the a trous method of Starck et al (citation needed). These were first included in the original scarlet in 2018 by Remy Joseph and were made more efficient in the transition to `scarlet_lite`, so much of the code that this PR is built from has been in `scarlet_lite` since it's inception. A complete description of the a trous method is beyond the scope of this technote and the reader is refered to the Starck citation above for a full description of the method.

What makes starlets particularly useful is that they are a form of compensated filter, meaning the wavelet kernel integrates to zero, so they act as a band pass filter. Compensated filters have been shown to be useful for source detection in astrophysical images (citations needed) although use of them has decreased, as work by Zackay (citations needed) showed that filtering by the PSF is a more optimal detection statistic for *isolated stats* under certain background conditions. However, with blending, galaxies, and the wide variety of backgrounds faced by LSST, we have found that detection with starlets improves our ability to be more background agnostic and importantly more sensitive to blended sources that detection misses when performed directly on PSF filtered coadds. An added side effect that we'll see later is a useful heuristic for associating some detections from shredded galaxies with a single structure (host galaxy), which helps remove spurious sources from the detection catalog.

Starlets work particularly well for detection because of two key features: the a trous method is fast and efficient and the starlet basis is actually a collection of compensated filters at different scales, allowing us to probe different features of galaxies. This multi-resolution (or multi-scale) detection enables us to find fainter low surface brightness (LSB) galaxies that the higher wavelet scales miss. At the time of this technote the final results are still being analyzed and it isn't yet clear if the tradeoff between detecting these LSB's is worth the potential spurious sources they they seem to be sensitive to. In the end we may end up using only a single wavelet scale.

#### Preparing images for detection

In order to have catalogs that can be merged properly we need to know the significance of each detection on a given *plane* (where a plane is defined as a unique `(band, scale)` combination). This involves taking the starlet coefficients at each scale and weighing them by the proper variance. Like `SourceDetectionTask`, users can choose between `pixel_stdev` and `stdev`, with the former being the default and recommended value just as in `SourceDetectionTask` and `DynamicDetectionTask`. However, in order to correctly calculate the standard deviation we need to properly weight the variance by the wavelet kernel at the appropriate scale. Once each band has been properly weighted into a significance map, we combine the significance map from all of the bands at the same scale into a `$\chi^2$`-like coadd by clipping them at zero and summing them together. This is slightly different from a `$\chi^2$` coadd in that it is convolved with the starlet kernel instead of the PSF and because we clip at zero. However, to simplify the language of this document we will still refer to this as a `$\chi^2$` coadd. The clip at zero has been shown to reduce outliers, as the `$\chi^2$` image is non-negative and large negative noise variations can occasionally have sufficient significance to register as detections. This does complicate the stated significance of the `$\chi^2$` coadd and the code goes through a procedure that is the equivalent of matching the image to chi^2 distribution and measuring the significance of each pixel in relation to a `$\chi^2$` distribution, however it is done in log space in order to account for the wide range of magnitudes of flux in an image that otherwise saturates the scipy methods that make the same measurements.

#### Detecting on significance maps

Once the significance maps are built, a watershed method (similar to SExtractor) is used to create footprints and peaks on each plane. A `$\kappa$` parameter is used to ensure that a sufficient saddle exists between two peak detections in order for them to be considered unique. Once peaks are detected on the same plane they are guaranteed to remain distinct and not merged by the merging algorithm. The result is a catalog of `candidates` that contains a row for each peak detection on each plane.

### Merging detections

While Kaiser (`$\chi^2$`) coadds have been shown to be optimal for detecting isolated point sources, blending in particular makes it useful to detect across multiple bands. In 2022 the science pipelines created a task to create `$\chi^2$` coadds and while it was an improvement over the `DynamicDetectionTask` results, it would often miss nearby sources that were blended together into a single detection on the `$\chi^2$` coadd while detecting on separate bands and then merging the results found a lot of visibly obvious blended sources that were easy to spot by eye. The new detection solves this by detecting in each `ugrizy` band *and* on the `$\chi^2$` coadd. For now it also detects at two different wavelet scales for a total of 14 detection images for each coadd, resulting the necessity for a more complicated procedure for merging detections across bands.

First we'll look at the current procedure in `MergeDetectionsTask`:
1. Rank the bands in priority order and iterate through the bands in order from highest to lowest priority.
2. Check to see if a footprint overlaps with any other footprint in the catalog (which it will not for the highest priority band).
3. For new footprints it adds the footprint and its peaks to the catalog.
4. For overlapping footprints:
  - the new footprint is the union of the existing footprint and the footprint for the band. This may result in merging of existing footprints.
  - It finds the nearest peak in the existing footprint to the peak in the incoming footprint. Any peaks closer than `0.3"` to another peak are dropped but the `merge_peak_<band>` flag is set to indicate that the peak was merged from another band. Peaks further than `1.0"` are added as new peaks to the existing footprint and any peaks between `0.3"` and `1.0"` are dropped without setting the `merge_peak_<band>` flag.
5. Sky sources are injected at locations that do not overlap with any existing footprints.
6. Peaks are culled based on a set of criteria to remove junk around big blends.

There are a few flaws in the above algorithm:
- The band priority rank does not take flux into account. This means that the position used from the highest priority band might have very low flux as opposed to another band that the source might be brighter in and thus have a more accurate position.
- A result of the above is that sometimes a peak will be culled because it was below the threshold in the highest priority band even though in another band it might have been bright enough to be considered.
- Nearby peaks in a single band might be merged into a single detection if there is no saddle between them. Lower priority bands that detect them as individual peaks can sometimes incorrectly merge the individual detections into a single peak.

The new MultiResolutionDetectionTask merging algorithm works differently. The basic outline (to be explained below) is
1. The candidates are squashed into a position catalog.
2. An adjacency graph with edges between positions that are within a search radius.
3. The adjacency graph is subdivided into cliques, where each clique is a single detected peak.
4. The unambiguous candidates are used to set the center of each peak using their median position.

#### Squashing candidates into positions

With detections made across 14 different planes and detections locations differing by at most a few pixels, candidates in different bands often end up with the same integer positions. To simplify the complexity of merging we first combine candidate peaks with the exact same `x` and `y` positions into a single positions catalog that uses bitmasks to track the band and wavelet scale for all candidates that were squashed together. For testing both the candidate and position catalogs are individual data products that we expect to be dropped before DR1 processing begins.

#### Building the adjacency graph

A search radius is created based on the maximum PSF FWHM across the bands and a graph is made that connects all pairs of objects that are within the search radius of each other. The exception is that neighboring peaks detected in the same plane, even within the search radius, are considered to be separate detections and do not have an edge in the adjacency matrix. Each connected sub-graph is called a component in the code. This is the same setup as a friends-of-friends algorithm, where individual components would be considered a single peak detection using that algorithm. This method differs in that friends of friends will merge two distinct peaks if one plane detected them as a single object and their midpoint is less than the detection radius away from each other (or guaranteed to be distinct based on same-plane detection). Instead, we treat the lack of an edge in the adjacency matrix to be evidence of distinct peaks, meaning that a peak in this scheme is a collection of positions that are all connected to each other by the graph, which in graph theory is called a clique. Determining the minimum number of cliques in a component is then the same as determining the distinct peaks created by a collection of nearby candidates.

#### From components to cliques

In theory this is an NP-hard problem but due to the low numbers of positions for each peak in the final catalog (clique) we can in practice solve this exactly in all but those most crowded fields. In those pathological cases a greedy algorithm is sufficient to find the exact solution nearly all of the time. The full merging procedure typically runs in O(`$\mu{s}$`) and at most just under `500ms` for the most crowded field that LSST was able to build coadds for in DP2.

#### Setting peak locations

In order to turn each clique into a peak we classify each position as an ambiguous or unambiguous detection. Ambiguous positions are ones that connect to all members of more than one clique. For example, midpoint detections can often be associated with the two peaks that were combined into a single detection in bands with a wider PSF and/or wavelet scale. For very nearby sources the majority of the detections can actually be midpoint detections so it's important to remove ambiguous detections first. Unless all of the members of a clique are ambiguous (which is rare but does happen, in which case all candidates are used), the median separable `x` and `y` of the unambiguous candidates is used to set the approximate center of the peak, and the candidate detection nearest to that location is chosen as the location for the peak.

### Associating shredded galaxies

One of the side effects of better detection of blended sources is that we now have a lot more galaxy sub-populations, for example star forming regions and spiral arms. In terms of deblending this is desirable: missing features with spectra that differ from the monotonic template created for a galaxy bulge results in poor deblending. In the long term we hope to train a NN classifier to identify sub-regions of a shredded galaxy that should be associated with a single structure as opposed to individual sources. In the meantime it was decided to implement a heuristic that identifies a large fraction of the shredded galaxies so that we can pass reasonable information down to the user to help them distinguish shredded galaxies from blended objects.

Luckily we can do this without much extra work from the significance maps that we already have. It was noticed while creating the detection algorithm that most sub-structure peaks that are part of shredded galaxies are all detected on the same footprint in the finest wavelet scale that we use for detection. Even stars in extreemly crowded fields rarely overlap in the same wavelet scale, and virtually never have more than two overlapping peaks on the same scale. On the other hand, extended objects like galaxies that have substructure peaks nearly always have 3 or more peaks detected at the same wavelet scale. This results in a useful heuristic: any peak in a starlet footprint with 2 or more other peaks is associated with other peaks by the same `structure_id`. This heuristic does occasionally flag blends that by chance have vert nearby galaxies, as well as stars and galaxies connected to bright, saturated stars. In the future (ie before this is turned on by default in the science pipelines) we plan to separately identify those peaks in pipe_tasks when the other blending flags are set by marking them as part of a structure with saturated pixels in the center.

For larger galaxies this is still only a subset of the larger galaxy, however galaxies sufficiently large enough and bright enough to be shredded even more than this heuristic detects happens less than once per tract and likely will not be addressed until a future classifier is created.

### Detecting Footprints

The footprints detected in (reference here) cannot be used downstream in the deblender or measurement algorithms because those footprints were made in the starlet band-limited significance maps. By eliminating the low frequency signal we are able to obtain a more accurate peak catalog but would miss the flux in most galaxies.

#### Footprints in DM-52721

The detection code created in this ticket is designed to be a drop in replacement for the current detection and merging pipeline tasks, which require footprints similar to the ones that are currently generated. Those are created in `MultiResolutionDetectionTask` using the same procedure as `SourceDetectionTask` with two key differences:
- Background estimation is never used.
- The footprints are detected directly on the `$\chi^2$` image.

This allows `MultiResolutionDetectionTask` to use the existing methods from `SourceDetectionTask` to generate footprints, clear their detection catalogs, and populate them with peaks that were detected in (reference here). Any peaks that fall outside of a footprint are dropped, as they are assumed to be spurious detections as they could not be associated with an above flux region in a combined footprint.

#### Footprints in the future

Before DR1 we intend to implement a rigorous procedure for splitting up full patch blends so that we no longer have to skip blends above a certain size and/or density threshold. That algorithm is mostly left for its own technote, however we outline it here as it will hopefully replace the above footprint detection scheme in the near future.

Currently the science pipelines detect peaks and footprints, then initialize blends, then deblend using `scarlet_lite`. In the future initialization and footprint detection will be switched. The idea is to create monotonic footprints on the partially deconvolved images at the location of each peak, which will make it easier for us to segment large blends into populations of smaller blends that will fit into memory during fitting. This eliminates the need to run an extra footprint detection on each tract/patch combination, as the footprints will instad be the merged footprints created by overlapping monotonic initial scarlet models.

## Analysis

Analysis will be left for a future ticket. The technote thus for is designed to aid the reviewer in understanding the PRs for DM-52721, which adds the code to perform multi-band detection but does not add as to any pipelines in `drp_pipe`. While this ticekt is being reviewed the analysis will be performed and this technote willbe updated with the results in the same ticket that implements the new multi-band detection as the default in the science pipelines step 3 processing. Analysis of its performance on single band images, where it is likely to avoid many of the hacks needed to process DP2 in crowded fields, will also be added to this technote in an even later ticket.
