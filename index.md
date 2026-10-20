# The New MultiBandDetectionTask

```{abstract}
Currently detection on coadds is performed running `DetectCoaddSourcesTask`, which runs `DynamicDetectionTask` on each band and then `MergeDetectionsTask` to merge the catalogs into a single catalog. `MultiBandDetectionTask` is a single pipeline task that replaces those calls by running `MultiResolutionDetectionTask` on all bands simultaneously, creating a more complete and pure catalog. This technote describes these new tasks and offers a comparison with the legacy `DynamicDetectionTask` results. It also lists the shortcomings of the new detection task and how they will be addressed in the future.
```

## Add content here

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.
