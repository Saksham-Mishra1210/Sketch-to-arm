# Sketch-to-Arm

A computer vision and robotics kinematics project: extracting a hand-drawn
shape from a photo and computing the inverse kinematics for a simulated
2-link robot arm to trace it.

## What it does

The pipeline takes a photo of a shape (a hand-drawn triangle, or a
photographed hand), extracts its outline, and simulates a 2-link robot arm
tracing that outline using inverse kinematics derived from scratch.

## Pipeline / Approach

1. **Image capture** — photo of a shape (drawn or photographed)
2. **Preprocessing** — grayscale, Gaussian blur, thresholding (Otsu's method,
   with the threshold direction chosen based on whether the subject is
   lighter or darker than its background)
3. **Contour extraction** — OpenCV's `findContours` traces the shape's outline
4. **Path simplification** — `cv2.approxPolyDP`, with an automated binary-search
   wrapper (`auto_simplify`) that finds the simplification tolerance (epsilon)
   needed to hit a target point-count range, rather than a hand-tuned fixed value
5. **Workspace scaling** — the extracted points are centered and scaled to fit
   the robot arm's reachable workspace
6. **Inverse kinematics** — for each point, the 2-link arm's joint angles
   (theta1, theta2) are computed using a closed-form solution (law of cosines
   + atan2), derived from scratch
7. **Animation** — the simulated arm is animated tracing the path, drawing a
   trail as it moves

## Key techniques and libraries

- **OpenCV** — image preprocessing, thresholding, contour extraction, path simplification
- **NumPy** — numerical operations, trigonometry, vectorized math
- **Matplotlib** — visualization and animation
- **Forward/Inverse Kinematics** — derived manually for a 2-link planar arm:
  - FK: x = L1·cos(θ1) + L2·cos(θ1+θ2), y = L1·sin(θ1) + L2·sin(θ1+θ2)
  - IK: solved via the law of cosines (for θ2) and atan2 (for θ1)
- **Path simplification (Douglas-Peucker / approxPolyDP)** — with an
  automated epsilon-selection step via binary search

## Results

- IK/FK round-trip accuracy: mean/max error ~1e-16 (floating-point precision
  noise), confirming the derived kinematics are mathematically exact
- Path simplification reduced a 195-point triangle contour to 5 meaningful
  points (the corners), while visually preserving the original shape
- Automated epsilon selection (`auto_simplify`) independently converged to
  the same result as manual tuning, and generalized to a more complex shape
  (a photographed hand) by adjusting the target point-count range
- Tested on two meaningfully different inputs: a hand-drawn triangle
  (straight edges, high contrast) and a photographed hand (curves, lower
  and uneven contrast) — the hand required a wider point-count target
  (89 points vs. 5) to accurately capture its curved finger shapes

## Limitations

- **Threshold direction is not automatic** — whether the subject is darker
  or lighter than its background must currently be known in advance
  (`THRESH_BINARY` vs `THRESH_BINARY_INV`). A fix was identified (comparing
  brightness of the image center vs. edges, since the subject is typically
  centered in the photo) but deliberately not implemented, prioritizing
  validation of the core pipeline on multiple test cases given project
  timeline constraints
- **Accuracy vs. motion smoothness trade-off** — simplifying the path to
  fewer points improves shape accuracy (sharp corners) but reduces motion
  smoothness (fewer intermediate steps); linear interpolation between
  simplified points was designed but not implemented
- **Fixed elbow configuration** — the arm always uses one fixed IK solution
  branch (elbow-up or elbow-down); near full extension this could cause
  abrupt joint changes, which a smarter branch-selection method could avoid
- **Low-contrast inputs are unreliable** — an initial hand photo against a
  similarly-toned background produced noisy/incomplete contours with both
  Otsu and adaptive thresholding; a high-contrast background was needed
- **2-link arm only** — the closed-form IK solution used here is specific
  to 2 links; more joints would require a different approach (e.g.
  numerical/iterative methods), since there's generally no clean closed-form
  solution beyond 2 links

## Development process

This project was built and debugged incrementally over several sessions.
A full log of bugs found, root causes, and fixes is documented in
`NOTES.md`, including coordinate-system mismatches, operator-precedence
errors, a thresholding-direction bug discovered while testing on a second
shape, and a path-simplification accuracy measurement.

## How to run

1. Open the notebook in Google Colab
2. Upload a photo of a shape (simple drawing or photographed object)
3. Run cells top to bottom
4. The final cell renders an animation of the simulated arm tracing the shape
5. Note: the animation output uses interactive JavaScript, which may not
render in GitHub's notebook preview. Open the notebook in Google Colab
to see it fully interactive, or view the static frame images included
alongside it.

## Author

Saksham Mishra (BT25MME102)
