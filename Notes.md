# Dev Log — Sketch-to-Arm

## Day 1 — Contour extraction setup

### Shadow interfering with thresholding
- Symptom: Otsu thresholding picked up a shadow in the photo as part of the "white" foreground region, since it was brighter than expected
- Fix: cropped the image to isolate just the triangle before running threshold/contour steps, avoiding the shadow region entirely
- Result: clean contour extraction — findContours found 9 total contours (triangle plus small noise), correctly selected the triangle as the largest by area


## Day 2 — FK/IK derivation and coding

### Derived FK and IK for a 2-link arm from scratch
- FK: x = L1·cos(θ1) + L2·cos(θ1+θ2), y = L1·sin(θ1) + L2·sin(θ1+θ2) — derived directly from arm geometry
- IK: solved using law of cosines (cos θ2 = (x²+y²−L1²−L2²)/(2·L1·L2)) to find θ2, then atan2 to find θ1 — derived and understood each step

### Bug: wrong trig identity in fk()
- Cause: wrote cos(theta1)*cos(theta2) instead of cos(theta1+theta2) — misunderstood which identity applied
- Fix: corrected to use the sum of angles inside the cosine, matching the derivation

### Bug: missing parentheses in ik()
- Cause: wrote np.arccos(x**2+y**2-l1**2-l2**2) / 2*l1*l2 — arccos only wrapped the numerator, and division/multiplication happened in the wrong order
- Fix: wrapped the full fraction in parentheses before passing it to arccos

### Bug: path_to_angles() only processed 1 point
- Cause: return angles was indented inside the for loop instead of after it, causing the function to exit after the first iteration
- Fix: corrected indentation so return runs only after the loop completes

### Bug: animated triangle trace appeared vertically flipped
- Cause: OpenCV's findContours returns points in image coordinates (y increases downward), while matplotlib and the FK/IK math use standard Cartesian coordinates (y increases upward)
- Fix: negated the y-values right after extracting contour points, before scaling: points[:, 1] = -points[:, 1]
- Lesson: combining computer vision (image coordinates) and robotics/math (Cartesian coordinates) requires an explicit coordinate-system conversion

### Verification
- Round-trip test (ik() then fk()) confirmed exact matches for test points
- Visual single-point test: plotted arm reaching (1,1) with theta1=0°, theta2=90° — tip landed exactly on target marker


## Day 3 — Accuracy measurement

### Added used_points tracking for index-safe accuracy checks
- Updated path_to_angles() to also return used_points, alongside angles, so indices stay aligned even if points are skipped as unreachable

### Measured IK/FK round-trip accuracy
- Result: mean/max error ~1e-16 — floating-point precision noise, confirming the IK/FK derivation is mathematically exact

### Bug: angle_path captured as a bundled tuple instead of unpacked
- Cause: angle_path = path_to_angles(workspace_points) only assigned one variable, but the function returns two — Python bundled both into a single tuple
- Fix: unpacked both return values explicitly

### Bug: cos_theta2 denominator missing parentheses (recurrence)
- Cause: same precedence bug as Day 2's IK fix had reappeared in path_to_angles(), masked since l1=l2=1.0
- Fix: added parentheses around the full denominator
- Lesson: a bug fixed in one function doesn't automatically get fixed everywhere the same logic is duplicated


## Day 4 — New photo test, path simplification, deviation measurement

### Re-tested pipeline on a second, independently-photographed triangle
- Bug: cv2.imread returned None because filename and crop coordinates were leftover from the previous photo
- Fix: verified img.shape directly, re-uploaded the correct file, re-tuned crop coordinates

### Implemented path simplification (cv2.approxPolyDP)
- Replaced arbitrary "every 20th point" sampling with a real simplification algorithm
- Used epsilon = 0.01 * perimeter — reduced contour from 195 points to 5 simplified points, visually accurate

### Bug: angle_path unpacking regression from a leftover duplicate cell
- Cause: an old, already-fixed cell was still present earlier in the notebook; running it after the corrected cell silently overwrote angle_path back into the broken state
- Fix: corrected the leftover cell, avoided re-running stale cells
- Lesson: duplicate cells touching the same variable can silently reintroduce old bugs if run out of order

### Trade-off discovered: shape accuracy vs. motion smoothness
- Raw/sampled contour: smooth motion, kinked/imprecise corners
- Simplified path: accurate sharp corners, choppier motion (fewer intermediate steps)
- Considered fix: linear interpolation between simplified points — scoped and designed but not implemented

### Added path deviation measurement (simplified vs. original contour)
- Bug: initial measurement gave suspiciously large values (mean 0.63, max 1.12) despite visual accuracy
  - Cause: scale_to_workspace() was called separately on each point set, each computing its own centroid/scale — misaligning the coordinate frames
  - Fix: computed centroid and max_dist once from the original contour, applied identically to both point sets (scale_with_params)
- Lesson: comparing two derived versions of the same data requires shared normalization parameters, not independently recomputed ones

### Noted for later: automating epsilon selection
- Manual tuning worked for the triangle but wouldn't generalize to a more complex shape
- Planned: binary-search epsilon based on a target point-count range


## Day 5 — Auto-epsilon implementation and palm/hand photo test

### Implemented auto_simplify() — automated epsilon selection via binary search
- Binary search over epsilon values, converging on whichever value produces a point count within a target range, replacing hardcoded epsilon values
- Verified: converged to the same 5-point result as manual tuning on the triangle, confirming correctness

### Testing pipeline on a second, harder shape (palm)
- First attempt: hand against a similarly-toned background (low contrast)
  - Otsu: noisy, spiky contour
  - adaptiveThreshold: found strong edges only between fingers (high local contrast there), incomplete elsewhere (low contrast along outer silhouette)
  - Conclusion: root cause was low contrast, not algorithm choice — both methods failed for the same underlying reason, differently
  - Kept adaptiveThreshold code commented out as a documented alternative for future low-contrast inputs

### Retake with high-contrast background
- Re-photographed hand against a dark, textured background — much better contrast
- Otsu threshold produced a clean, solid hand silhouette

### Bug: findContours only found a 4-point contour tracing the image frame, not the hand
- Diagnosis: printed area of every contour — confirmed area matched the hand's expected size, but point count (4) was suspiciously low
- First fix attempt: CHAIN_APPROX_SIMPLE -> CHAIN_APPROX_NONE — increased points to 5996, but still traced the image border
- Root cause: checked corner pixel values of thresh directly — all corners were white, confirming the entire image had a white border/frame being traced instead of the hand
- Real root cause: THRESH_BINARY_INV was being reused from the triangle pipeline, but this hand photo had a dark background and lighter hand (reversed relationship) — the inverted threshold made the background white and the hand black
- Fix: switched to plain THRESH_BINARY for this photo
- Lesson: threshold direction must match whether the subject is darker or lighter than its background — not universal, doesn't automatically generalize across photos. A fix was identified (compare brightness of image center vs. edges to auto-detect direction) but deliberately not implemented, prioritizing validation of the core pipeline given timeline constraints

### Simplification results on the hand
- Initial attempt: auto_simplify(largest, min_points=15, max_points=40) reduced ~6000 points to ~20 points — recognizable hand structure, but individual fingers appeared as sharp zigzag spikes due to limited point budget per finger
- Improved attempt: widened the range to auto_simplify(largest, min_points=50, max_points=500) — converged to epsilon=5.876, giving 89 points after 6 iterations
- Result: significantly better visual quality — fingers appear smoothly rounded with distinct, recognizable shapes (including the thumb), and the palm curve is well-preserved
- Lesson: the appropriate point-count target depends on the shape's real complexity — a hand's curved fingers need substantially more points than a triangle's straight edges. auto_simplify's value is that this can be tuned via min_points/max_points, without guessing a new epsilon multiplier per shape

### Bug: y-flip fix dropped in later pipeline rebuilds
- The y-flip fix (points[:, 1] = -points[:, 1]) was applied and verified working in Day 2
- In later sessions, after switching between largest, simplified, and simplified_auto as the pipeline's point source across multiple rebuilds and session resets, the y-flip line was not consistently re-added to every version
- Fix: re-applied the y-flip to all pipeline-rebuild cells, regardless of which point source is actively used
- Lesson: a fix applied once isn't automatically preserved when the pipeline is restructured across sessions — worth periodically re-verifying previously-fixed bugs are still actually in effect

### Key general lesson from this session
- A pipeline tuned on one input carries hidden assumptions (threshold direction, contrast requirements, point-count needs) that don't automatically hold for a different kind of input — testing on a second, meaningfully different shape surfaced real bugs and design trade-offs the triangle alone never would have revealed
