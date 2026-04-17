# prepipe-containers

Container images for the E11 preprocessing pipeline.

## Images

### bigstitcher-spark

BigStitcher-Spark with concurrent tile optimization enabled. Based on
[JaneliaSciComp/BigStitcher-Spark](https://github.com/JaneliaSciComp/BigStitcher-Spark)
with patches from:

- [e11bio/mpicbg](https://github.com/e11bio/mpicbg/tree/prepipe) — Fix
  `TileUtil.optimizeConcurrently()` crash on disconnected tiles
- [e11bio/multiview-reconstruction](https://github.com/e11bio/multiview-reconstruction/tree/prepipe)
  — Re-enable concurrent solver in `GlobalOptIterative`

```bash
# Pull the image
docker pull ghcr.io/e11bio/bigstitcher-spark:latest

# Use in Nextflow (apptainer)
apptainer.runOptions = ''  # no bind mount needed
process {
    withName:BIGSTITCHER_MODULE {
        container = 'ghcr.io/e11bio/bigstitcher-spark:latest'
    }
}
```

Built automatically on push to `main` via GitHub Actions.
