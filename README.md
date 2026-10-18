# Discontinuity and Trace

Python notebooks for identifying rock discontinuity surfaces and extracting three-dimensional traces from LAS point clouds.

## Notebooks

- [`las_hough_normals_open3d.ipynb`](las_hough_normals_open3d.ipynb): local Hough-style normal estimation, LAS export with normal fields, and Open3D visualization.
- [`discontinuity_trace_extraction.ipynb`](discontinuity_trace_extraction.ipynb): discontinuity and edge separation, spatial clustering, and trace extraction through the final 3D Shapefile export. Subsequent spherical-window statistics and evaluation experiments are excluded.

## Algorithm steps

1. **Estimate local normals.** Query 30 nearest neighbors and vote over 512 Fibonacci-sphere directions using distance-weighted orthogonality scores. Save the selected unit normals as LAS Extra Bytes fields.
2. **Separate surfaces and edges.** Compute the mean unsigned angular difference between neighboring normals. Apply three-class fuzzy c-means (FCM); the class with the lowest curvature center supplies plane candidates, and the remaining classes supply edge candidates.
3. **Cluster the candidates.** Apply DBSCAN separately to edge points (`eps=0.12`, `min_samples=4`) and plane candidates (`eps=0.07`, `min_samples=4`). Retain non-noise edge points for trace extraction.
4. **Estimate surface orientations.** Use PCA to estimate the orientation of eligible DBSCAN plane clusters. Export the colored plane point cloud and an orientation spreadsheet.
5. **Reconstruct traces.** Associate each cleaned edge point with its two dominant neighboring plane labels using 60 neighbors. Use the cross product of the fitted PCA normals for the trace direction and the grouped edge-point centroid for its position. Project points onto this line and split segments at gaps greater than `0.5` coordinate units. The final trace stage uses the original DBSCAN plane labels.
6. **Save the result.** Export trace endpoints as a `POLYLINEZ` Shapefile with segment ID and three-dimensional length.

## Run

Use Python 3.10 or later and install the dependencies:

```bash
python -m pip install jupyterlab numpy scipy pandas matplotlib scikit-learn scikit-fuzzy open3d laspy joblib tqdm pyshp openpyxl
jupyter lab
```

Run from the repository folder. An example LAS point cloud is included at [`data/point_cloud.las`](data/point_cloud.las); open the normal-estimation notebook and run its cells in order. It creates `outputs/point_cloud_hough_normals_k30.las` and a JSON sidecar.

The extraction notebook reads a space-delimited normal table at `data/point_cloud_normals.txt`; its last three columns must contain one normal per input point, in exactly the same order. Convert the companion notebook's LAS output with:

```python
import laspy
import numpy as np

cloud = laspy.read("outputs/point_cloud_hough_normals_k30.las")
normals = np.column_stack((cloud.normal_x, cloud.normal_y, cloud.normal_z))
np.savetxt("data/point_cloud_normals.txt", normals, fmt="%.8f")
```

Then run the extraction notebook in order. Open3D windows are interactive: close each window to continue execution. The normal notebook exposes `RUN_VISUALIZATION` for its final display.

## Outputs and conventions

- Main outputs are written under `outputs/`, including `trace_lines_3d.shp`, `.shx`, and `.dbf`. Keep these Shapefile components together.
- Distance parameters use the input coordinate units. The extraction notebook subtracts the input coordinate minima. The final export retains the source default `global_offset=None`, so traces use this local coordinate frame. To restore input coordinates, pass `offset` to the final export call as `global_offset=offset`; assign the input CRS separately when needed.
- Hough normals are unoriented axes: `n` and `-n` are equivalent. Consistent orientation is needed when interpreting the sign-sensitive ridge/valley classification.
- The FCM, DBSCAN, and trace-extraction parameters are retained, with portable paths and explicit definitions of the FCM non-plane edge mask and its colors. Saved execution outputs are cleared.
