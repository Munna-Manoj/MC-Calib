
# MC-Calib

Toolbox described in the paper ["MC-Calib: A generic and robust calibration toolbox for multi-camera systems"](https://www.sciencedirect.com/science/article/abs/pii/S1077314221001818) ([RG](https://www.researchgate.net/publication/357801965_MC-Calib_A_generic_and_robust_calibration_toolbox_for_multi-camera_systems) for open access, [preprint](https://github.com/rameau-fr/MC-Calib/issues/4)).

**This fork adds Double Sphere (DS) camera model support** for calibrating heterogeneous multi-camera systems (e.g., Brown + Double Sphere + Brown).

![](docs/illustration.png)

---

# Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration Reference](#configuration-reference)
  - [Board Parameters](#board-parameters)
  - [Camera Parameters](#camera-parameters)
  - [Image Parameters](#image-parameters)
  - [Optimization Parameters](#optimization-parameters)
  - [Output Parameters](#output-parameters)
- [Data Directory Structure](#data-directory-structure)
- [Pre-calibrated Intrinsics File](#pre-calibrated-intrinsics-file)
- [Supported Camera Models](#supported-camera-models)
- [Common Scenarios](#common-scenarios)
- [Output Files](#output-files)
- [Troubleshooting](#troubleshooting)
- [Citation](#citation)

---

# Installation

For Windows users, follow [this installation guide](/docs/Windows.md).

Requirements: Ceres, Boost, OpenCV {4.2.0, 4.5.5, 4.10.0, 4.11.0}, c++17

- [Install](https://docs.docker.com/engine/install/) docker

- Pull the image:

   ```bash
   docker pull bailool/mc-calib-prod:opencv4110 # production environment
   docker pull bailool/mc-calib-dev:opencv4110  # development environment
   ```

- Run pulled image (set `PATH_TO_REPO_ROOT` and `PATH_TO_DATA` appropriately):

   ```bash
   docker run \
               -ti --rm \
               --volume="$PATH_TO_REPO_ROOT:/home/MC-Calib" \
               --volume="$PATH_TO_DATA:/home/MC-Calib/data" \
               bailool/mc-calib-prod:opencv4110
   ```

Compiling the code:

   ```bash
   cd MC-Calib
   mkdir build
   cd build
   cmake -DCMAKE_BUILD_TYPE=Release ..
   make -j10
   ```

Documentation is available [online](https://codedocs.xyz/rameau-fr/MC-Calib/). To generate local documentation, follow [the instructions](/docs/Documentation.md).

---

# Quick Start

```bash
# 1. Prepare your config file (see Configuration Reference below)
# 2. Run calibration
./apps/calibrate/calibrate path/to/your_config.yml

# 3. (Optional) Post-calibration analysis
python3 python_utils/post_calibration_analysis.py -d path/to/save_path
```

---

# Configuration Reference

The configuration file is a YAML file that controls every aspect of the calibration. Below is a **complete field-by-field reference** with explanations, valid values, and tips.

## Board Parameters

These fields describe the ChArUco calibration boards you printed and are using.

```yaml
######################################## Board Parameters ########################################

number_board: 2
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `number_board` | int | Yes | **How many distinct ChArUco boards you are using.** Each board has a unique set of ArUco marker IDs. If you printed 2 different boards, set this to `2`. Even if only 1 board appears in most frames, set this to the total number of distinct boards in your setup. |

```yaml
boards_index: []
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `boards_index` | list of int | Yes | **Which board IDs to detect.** Leave empty `[]` if your boards have sequential IDs starting from 0 (i.e., board 0, board 1, ..., board N-1). If you only want to detect specific boards (e.g., boards 5 and 10 out of many), use `[5, 10]`. This is useful when you have many printed boards but only a few appear in your calibration images -- it speeds up detection by skipping boards that aren't present. |

> **How board IDs work:** Each ChArUco board is assigned a unique range of ArUco marker IDs. Board 0 gets the first batch, board 1 gets the next batch, etc. The `boards_index` tells MC-Calib which boards to look for. The marker ID ranges are determined by the board dimensions and dictionary size.

```yaml
number_x_square: 7
number_y_square: 5
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `number_x_square` | int | Yes | **Number of squares in the horizontal (X) direction** of your ChArUco board. Count all squares across one row, including both black and white squares. For a 7x5 board, this is `7`. |
| `number_y_square` | int | Yes | **Number of squares in the vertical (Y) direction.** Count all squares down one column. For a 7x5 board, this is `5`. |

> **Important:** These are the number of **squares**, not the number of inner corners. A 7x5 square board produces a 6x4 grid of inner corners (one fewer in each direction).

```yaml
length_square: 0.04
length_marker: 0.03
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `length_square` | float | Yes | **Ratio parameter for ChArUco board generation.** This controls the relative size of the checkerboard square when generating the board image. For standard boards, `0.04` works well. This does NOT need to match the physical printed size -- it's a generation parameter only. |
| `length_marker` | float | Yes | **Ratio parameter for the ArUco marker inside each square.** Must be smaller than `length_square`. For standard boards, `0.03` works well. The ratio `length_marker / length_square` determines how much of each square the ArUco marker fills. |

> **Common confusion:** `length_square` and `length_marker` are NOT the physical measurements of your printed board. They are parameters for the ChArUco board generator and ArUco detector. Keep them at `0.04` and `0.03` respectively unless you generated your boards with different values.

```yaml
square_size: 54
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `square_size` | float | Yes | **Physical size of one square on your printed board**, measured with a ruler. The unit can be anything (mm, cm, inches) -- all calibration results will be in the same unit. For example, if each square is 54mm, set `square_size: 54` and your extrinsics will be in mm. |

> **Tip:** Measure accurately! This directly affects the scale of your extrinsic calibration. Use calipers if possible.

### Per-Board Overrides (for boards of different sizes)

If all your boards have the same dimensions and square size, leave these empty:

```yaml
number_x_square_per_board: []
number_y_square_per_board: []
square_size_per_board: []
resolution_x_per_board: []
resolution_y_per_board: []
```

If boards differ, specify per-board values. Example with 3 differently-sized boards:

```yaml
number_board: 3
number_x_square_per_board: [5, 7, 6]
number_y_square_per_board: [7, 4, 6]
square_size_per_board: [14, 13.8, 16.3]       # physical square size per board
resolution_x_per_board: [1000, 500, 1000]      # board image generation resolution
resolution_y_per_board: [1000, 500, 1000]
```

| Field | Type | Description |
|-------|------|-------------|
| `number_x_square_per_board` | list | X-squares for each board. Length must equal `number_board`. |
| `number_y_square_per_board` | list | Y-squares for each board. Length must equal `number_board`. |
| `square_size_per_board` | list | Physical square size for each board. |
| `resolution_x_per_board` | list | Horizontal pixel resolution for generated board image. |
| `resolution_y_per_board` | list | Vertical pixel resolution for generated board image. |

> When per-board arrays are provided, they override `number_x_square`, `number_y_square`, and `square_size`.

---

## Camera Parameters

```yaml
number_camera: 3
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `number_camera` | int | Yes | **Total number of cameras in your rig.** MC-Calib expects exactly this many camera folders in `root_path`. |

```yaml
distortion_model: 0
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `distortion_model` | int | Yes | **Default distortion model for all cameras.** `0` = Brown (standard perspective, 5 distortion coefficients), `1` = Kannala-Brandt (fisheye, 4 coefficients), `2` = Double Sphere (wide-angle/fisheye, 2 parameters: xi, alpha). This is used for all cameras unless `distortion_per_camera` overrides it. |

```yaml
distortion_per_camera: [0, 2, 0]
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `distortion_per_camera` | list of int | No | **Per-camera distortion model override.** Leave empty `[]` if all cameras use the same model (specified by `distortion_model`). For heterogeneous systems, specify the model for each camera. Length must equal `number_camera`. Example: `[0, 2, 0]` means camera 0 = Brown, camera 1 = Double Sphere, camera 2 = Brown. |

> **Supported model IDs:**
> | ID | Model | Parameters | Best For |
> |----|-------|-----------|----------|
> | `0` | Brown | k1, k2, p1, p2, k3 | Standard perspective cameras |
> | `1` | Kannala-Brandt | k1, k2, k3, k4 | Fisheye cameras |
> | `2` | Double Sphere | xi, alpha | Wide-angle / fisheye cameras (closed-form inverse) |

```yaml
refine_corner: 1
min_perc_pts: 0.5
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `refine_corner` | int (0 or 1) | Yes | **Enable sub-pixel corner refinement.** `1` = enabled (recommended), `0` = disabled. Improves accuracy but slightly slower. |
| `min_perc_pts` | float (0.0-1.0) | Yes | **Minimum percentage of board corners that must be visible** for a detection to be considered valid. `0.5` means at least 50% of the board's corners must be detected. Lower this (e.g., `0.3`) if cameras have limited overlap and can only see part of the board. |

```yaml
cam_params_path: "None"
fix_intrinsic: 0
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cam_params_path` | string | Yes | **Path to a YAML file with pre-calibrated intrinsics.** Set to `"None"` if you want MC-Calib to estimate intrinsics from scratch. If you already know your camera intrinsics (from a separate calibration), provide the path here. See [Pre-calibrated Intrinsics File](#pre-calibrated-intrinsics-file) for the format. |
| `fix_intrinsic` | int (0 or 1) | Yes | **Freeze intrinsics during optimization.** `0` = intrinsics are estimated and refined (default). `1` = intrinsics from `cam_params_path` are loaded and held fixed -- only extrinsics are optimized. **You MUST provide `cam_params_path` when `fix_intrinsic: 1`.** |

> **When to use `fix_intrinsic: 1`:**
> - When you have high-quality pre-calibrated intrinsics (e.g., from a dedicated single-camera calibration with many images)
> - When using the Double Sphere model -- the DS heuristic initialization (fx = 0.8 * image_width) may not converge well for all lenses. Pre-calibrating DS intrinsics separately and freezing them here gives much better results
> - When you only care about extrinsic calibration (relative camera poses)

---

## Image Parameters

```yaml
root_path: "../data/my_images"
cam_prefix: "cam_"
keypoints_path: "None"
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `root_path` | string | Yes | **Path to the root directory containing camera image folders.** Can be relative (to the build directory) or absolute. This directory must contain exactly `number_camera` subdirectories named `{cam_prefix}001`, `{cam_prefix}002`, etc. |
| `cam_prefix` | string | Yes | **Prefix for camera folder names.** Default is `"Cam_"`. The folders must be named `{cam_prefix}001`, `{cam_prefix}002`, ..., `{cam_prefix}NNN`. |
| `keypoints_path` | string | No | **Path to a previously saved keypoints file** for faster re-runs. Set to `"None"` or `""` on first run. After the first run, MC-Calib saves detected keypoints; point this to that file to skip re-detection. |

> **Camera numbering is 1-based!** Folders must start from `001`, not `000`. For 3 cameras with `cam_prefix: "cam_"`:
> ```
> root_path/
>   cam_001/    # First camera images
>     000000.png
>     000001.png
>     ...
>   cam_002/    # Second camera images
>     000000.png
>     000001.png
>     ...
>   cam_003/    # Third camera images
>     000000.png
>     000001.png
>     ...
> ```

> **Image filenames must be identical** across camera folders. MC-Calib matches frames by filename (sorted alphabetically). If cam_001 has `000000.png`, cam_002 and cam_003 must also have `000000.png` for that frame.

---

## Optimization Parameters

```yaml
quaternion_averaging: 1
ransac_threshold: 10
number_iterations: 1000
he_approach: 0
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `quaternion_averaging` | int (0 or 1) | Yes | **Rotation averaging method.** `1` = Quaternion averaging (recommended), `0` = Median rotation. |
| `ransac_threshold` | float | Yes | **RANSAC inlier threshold in pixels** for initial pose estimation. Higher values are more tolerant of outliers. `10` is a good default. Lower to `3-5` if you have very precise detections and want to be stricter. |
| `number_iterations` | int | Yes | **Maximum iterations for Ceres non-linear refinement.** `1000` is usually sufficient. Increase if the solver reports it didn't converge. |
| `he_approach` | int (0 or 1) | Yes | **Hand-eye calibration approach.** `0` = Bootstrapped technique (recommended, more robust), `1` = Traditional approach. |

---

## Output Parameters

```yaml
save_path: "../results"
save_detection: 1
save_reprojection: 1
camera_params_file_name: ""
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `save_path` | string | Yes | **Directory where all output files are saved.** Created automatically if it doesn't exist. |
| `save_detection` | int (0 or 1) | Yes | **Save detection visualization images.** `1` = save images showing detected ChArUco corners overlaid on each frame. Useful for debugging detection issues. |
| `save_reprojection` | int (0 or 1) | Yes | **Save reprojection visualization images.** `1` = save images showing reprojected points vs detected points. Essential for visually verifying calibration quality. |
| `camera_params_file_name` | string | No | **Custom filename for the output camera parameters.** Leave empty `""` to use the default `calibrated_cameras_data.yml`. Set to e.g., `"camera_params.yml"` for a custom name. |

---

# Data Directory Structure

Here is the expected directory layout for a 3-camera system:

```
root_path/                          # Pointed to by root_path in config
  cam_001/                          # Camera 1 images (prefix + 3-digit index)
    000000.png                      # Frame 0
    000001.png                      # Frame 1
    000002.png                      # ...
    ...
  cam_002/                          # Camera 2 images
    000000.png                      # Same filenames as cam_001
    000001.png
    ...
  cam_003/                          # Camera 3 images
    000000.png
    000001.png
    ...
  precalibrated_intrinsics.yml      # (Optional) Pre-calibrated intrinsics

save_path/                          # Created by MC-Calib
  calibrated_cameras_data.yml       # Camera intrinsics + extrinsics
  calibrated_objects_data.yml       # 3D board structure
  calibrated_objects_pose_data.yml  # Board poses per frame
  reprojection_error_data.yml       # Per-corner reprojection errors
  convergence_00X.png               # (if save_reprojection: 1) reprojection images
  convergence_detection_00X.png     # (if save_detection: 1) detection images
```

> **Key rules:**
> - Camera folders use **3-digit** indices starting from **001** (not 000)
> - All camera folders must have **matching filenames** for synchronized frames
> - Images can be `.png`, `.jpg`, `.bmp`, etc. (anything OpenCV can read)
> - The `cam_prefix` in config must match the folder names (e.g., `"Cam_"` -> `Cam_001/`)

---

# Pre-calibrated Intrinsics File

When using `fix_intrinsic: 1` or providing initial intrinsics via `cam_params_path`, the file must follow this OpenCV YAML format:

### Brown (Perspective) Camera

```yaml
camera_0:
   camera_matrix: !!opencv-matrix
      rows: 3
      cols: 3
      dt: d
      data: [ fx, 0., cx,
              0., fy, cy,
              0., 0., 1. ]
   distortion_vector: !!opencv-matrix
      rows: 1
      cols: 5
      dt: d
      data: [ k1, k2, p1, p2, k3 ]
```

### Double Sphere Camera

```yaml
camera_1:
   camera_matrix: !!opencv-matrix
      rows: 3
      cols: 3
      dt: d
      data: [ fx, 0., cx,
              0., fy, cy,
              0., 0., 1. ]
   distortion_vector: !!opencv-matrix
      rows: 1
      cols: 2
      dt: d
      data: [ xi, alpha ]
```

### Kannala-Brandt (Fisheye) Camera

```yaml
camera_2:
   camera_matrix: !!opencv-matrix
      rows: 3
      cols: 3
      dt: d
      data: [ fx, 0., cx,
              0., fy, cy,
              0., 0., 1. ]
   distortion_vector: !!opencv-matrix
      rows: 1
      cols: 4
      dt: d
      data: [ k1, k2, k3, k4 ]
```

> **Camera indices are 0-based** in the intrinsics file (`camera_0`, `camera_1`, ...) even though image folders are 1-based (`cam_001`, `cam_002`, ...). `camera_0` corresponds to `cam_001`.

> **DS parameter xi** must be in range [-1, 1] and **alpha** in [0, 1].

---

# Supported Camera Models

## Brown (Perspective) -- `distortion_model: 0`

Standard pinhole camera with radial and tangential distortion. 5 distortion parameters: `k1, k2, p1, p2, k3`.

**Best for:** Standard cameras, webcams, industrial cameras with moderate field of view.

## Kannala-Brandt (Fisheye) -- `distortion_model: 1`

Equidistant fisheye model with 4 parameters: `k1, k2, k3, k4`.

**Best for:** Fisheye cameras with OpenCV's fisheye module calibration.

## Double Sphere -- `distortion_model: 2`

Compact 6-parameter model (fx, fy, cx, cy, xi, alpha) with a closed-form inverse. From [Usenko et al. 2018](https://arxiv.org/abs/1807.08957).

**Best for:** Wide-angle and fisheye cameras. Advantages over Kannala-Brandt:
- Closed-form unprojection (no iterative solver needed)
- Only 2 distortion parameters (xi, alpha) -- less overfitting risk
- Smooth projection across the entire field of view

**DS parameter guide:**
| Parameter | Range | Meaning |
|-----------|-------|---------|
| `xi` | [-1, 1] | Controls the shape of the first sphere. `xi=0` reduces to a single-sphere model. |
| `alpha` | [0, 1] | Blending between the two spheres. `alpha=0` reduces to pinhole. `alpha=0.5` is the transition point. |

---

# Common Scenarios

## Scenario 1: Simple stereo (2 perspective cameras)

```yaml
number_camera: 2
number_board: 1
distortion_model: 0
distortion_per_camera: []       # both cameras use Brown
fix_intrinsic: 0                # estimate intrinsics
cam_params_path: "None"
```

## Scenario 2: Heterogeneous stereo (perspective + fisheye)

```yaml
number_camera: 2
distortion_model: 0
distortion_per_camera: [0, 1]   # cam1=Brown, cam2=Kannala
fix_intrinsic: 0
cam_params_path: "None"
```

## Scenario 3: Heterogeneous 3-camera (Brown + Double Sphere + Brown)

```yaml
number_camera: 3
distortion_model: 0
distortion_per_camera: [0, 2, 0]   # Brown, DS, Brown
fix_intrinsic: 1                    # freeze pre-calibrated intrinsics
cam_params_path: "path/to/precalibrated_intrinsics.yml"
```

> **Recommended for DS cameras:** Pre-calibrate intrinsics separately and use `fix_intrinsic: 1`. The DS heuristic initialization may not converge for all lenses.

## Scenario 4: Extrinsic-only calibration (known intrinsics)

```yaml
fix_intrinsic: 1
cam_params_path: "path/to/known_intrinsics.yml"
```

## Scenario 5: Non-overlapping cameras with multiple boards

```yaml
number_board: 3
boards_index: []                   # detect all 3 boards
he_approach: 0                     # bootstrapped hand-eye
number_camera: 4
```

> **For non-overlapping cameras:** You need multiple boards visible simultaneously in different cameras. MC-Calib uses hand-eye calibration to chain the transforms.

---

# Output Files

After a successful calibration, MC-Calib generates these files in `save_path`:

### `calibrated_cameras_data.yml` (or custom name)

Contains intrinsics and extrinsics for every camera:

```yaml
nb_camera: 3
camera_0:
   camera_matrix: ...        # 3x3 intrinsic matrix [fx, 0, cx; 0, fy, cy; 0, 0, 1]
   distortion_vector: ...    # distortion coefficients (5 for Brown, 4 for Kannala, 2 for DS)
   distortion_type: 0        # 0=Brown, 1=Kannala, 2=Double Sphere
   camera_group: 0           # should be 0 if calibration succeeded
   img_width: 1280
   img_height: 800
   camera_pose_matrix: ...   # 4x4 transformation matrix (camera 0 = identity = reference)
```

> **camera_0 is always the reference frame** with an identity pose matrix. All other cameras' poses are expressed relative to camera_0.

> **camera_group** indicates connected components. If all cameras are in group 0, the full chain of transforms was successfully estimated. Different group numbers mean some cameras couldn't be linked.

### `calibrated_objects_data.yml`

The refined 3D structure of each calibration board.

### `calibrated_objects_pose_data.yml`

The 6-DOF pose (Rodrigues rotation + translation) of each board in every frame where it was detected.

### `reprojection_error_data.yml`

Per-corner, per-camera, per-frame reprojection errors for detailed analysis.

---

# Troubleshooting

### "No boards detected in camera X"
- Check that your board dimensions (`number_x_square`, `number_y_square`) match the physical board
- Ensure `length_square` and `length_marker` match the values used when generating the board
- If using multiple boards, check that `boards_index` is set correctly
- Enable `save_detection: 1` to visually inspect what's being detected

### "Calibration diverges / huge reprojection error"
- For DS cameras: use `fix_intrinsic: 1` with pre-calibrated intrinsics
- Lower `min_perc_pts` if boards are only partially visible
- Check that `square_size` is accurate (wrong scale = wrong extrinsics)
- Try increasing `number_iterations`

### "Camera X is in a different camera_group"
- This means MC-Calib couldn't find a chain of co-visible boards linking this camera to the others
- Solution: use more boards, or position boards so they're visible in overlapping camera pairs
- For non-overlapping setups, use `he_approach: 0` with multiple boards

### "Assertion failed: nb_camera > 0" or "nb_board > 0"
- Check your YAML syntax -- OpenCV's YAML parser is strict
- Ensure the file starts with `%YAML:1.0` and `---`

### "No such file or directory: cam_001"
- Camera folders must be 1-indexed (`cam_001`, not `cam_000`)
- Check that `cam_prefix` matches your folder names exactly
- Check that `root_path` is correct relative to the build directory

### DS camera: "Images appear mirrored"
- Some cameras (e.g., certain fisheye or special optics) produce horizontally flipped images
- Pre-flip images with `cv2.flip(img, 1)` before running MC-Calib
- Remember to flip the `cx` value in pre-calibrated intrinsics: `cx_flipped = image_width - 1 - cx_original`

---

# Contribution

Please follow `docs/contributing.rst` when introducing changes.

# Datasets
The synthetic and real datasets acquired for this paper are freely available via the following links:
- [Real Data](https://drive.google.com/file/d/143jdSi5fxUGj1iEGbTIQPfSqcOyuW-MR/view?usp=sharing)
- [Synthetic Data](https://drive.google.com/file/d/1CxaXUbO4E9WmaVrYy5aMeRLKmrFB_ARl/view?usp=sharing)

# Citation

If you use this project in your research, please cite:
```
@article{RAMEAU2022103353,
title = {MC-Calib: A generic and robust calibration toolbox for multi-camera systems},
journal = {Computer Vision and Image Understanding},
pages = {103353},
year = {2022},
issn = {1077-3142},
doi = {https://doi.org/10.1016/j.cviu.2021.103353},
url = {https://www.sciencedirect.com/science/article/pii/S1077314221001818},
author = {Francois Rameau and Jinsun Park and Oleksandr Bailo and In So Kweon},
keywords = {Camera calibration, Multi-camera system},
}
```
