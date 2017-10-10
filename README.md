## P4 Writeup -- Reinhard Steffens

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image_calib]: ./pics/output_4_5.png "Calibration"
[image_calib_test]: ./pics/output_6_0.png "Calibration Test"

[thr_test_01]: ./pics/output_15_0.png "Threshold Test 01"
[thr_test_02]: ./pics/output_15_1.png "Threshold Test 02"
[thr_test_03]: ./pics/output_15_2.png "Threshold Test 03"
[thr_test_04]: ./pics/output_15_3.png "Threshold Test 04"

[warp_test_01]: ./pics/output_21_1.png "Warp Test 01"
[warp_test_02]: ./pics/output_21_3.png "Warp Test 02"
[warp_test_03]: ./pics/output_21_7.png "Warp Test 03"
[warp_test_04]: ./pics/output_21_11.png "Warp Test 04"

[thr_warp_test_01]: ./pics/output_25_1.png "Threshold and Warp Test 01"
[thr_warp_test_02]: ./pics/output_25_3.png "Threshold and Warp Test 02"
[thr_warp_test_03]: ./pics/output_25_7.png "Threshold and Warp Test 03"
[thr_warp_test_04]: ./pics/output_25_9.png "Threshold and Warp Test 04"

[lane_find_test_01]: /pics/output_31_3.png "Lane Find Test 01"
[lane_find_test_02]: /pics/output_31_5.png "Lane Find Test 02"
[lane_find_test_03]: /pics/output_31_9.png "Lane Find Test 03"
[lane_find_test_04]: /pics/output_31_11.png "Lane Find Test 04"

[pipeline_test_01]: /pics/output_35_0.png "Pipeline Test 01"

[video]: ./out_test.mp4 "Video"

##Camera Calibration

The relevant code can be found in the notebook "./P4.ipynb" in the section titled "Camera Calibration"  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

Here is an example of an image used for the calibration including the detected corners:

![alt text][image_calib]

The output `objpoints` and `imgpoints` are then stored for the distortion correction step.

##Pipeline (single images)

###1. Distortion Correction.
I used the `objpoints` and `imgpoints` from the previous calibration and obtained the distortion coefficients using `cv2.calibrateCamera()`.  Here is an example of an original image and the undistorted version obtained by using `cv2.undistort()`

![alt text][image_calib_test]

###2. Applying Color and Gradient Threshold

The relevant code can be found in the notebook "./P4.ipynb" in the section titled "Color Gradient Threshold"

After some experiments I decided to use a combination of 'S' channel and 'x-gradient', similar to the method in the lecture.  The best thresholds for me where

```python
sx_thresh = (40 , 100)
s_thresh  = (180, 255)
```

Here are four example images.  Each is shown in the original, the color and gradient channel components (in green and blue), and the combination of both channels as gray-scale image:

![alt text][thr_test_01]
![alt text][thr_test_02]
![alt text][thr_test_03]
![alt text][thr_test_04]

###3. Perspective Transform

The relevant code can be found in the notebook "./P4.ipynb" in the section titled "Perspective Transform"

I chose the source and destination points in the following manner (eye-balling the specific values):

```python
src = np.float32([
        [570, 450],
        [710, 450],
        [100, 660],
        [1180, 660]])

dst = np.float32([
        [0,0],
        [1280, 0],
        [0, 720],
        [1280, 720]])
```

To compute the transformation matrix I used

```python
M = cv2.getPerspectiveTransform(src, dst)
```

Note: At this point I also saved the inverse transformation `Minv` which will be applied later in the video pipeline.

This is then applied to an undistorted example picture using

```python
cv2.warpPerspective(undist, M, (img.shape[1], img.shape[0]), flags=cv2.INTER_LINEAR)
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and plotted its warped counterpart to verify that the lines appear parallel in the warped image:

![alt text][warp_test_01]
![alt text][warp_test_02]
![alt text][warp_test_03]
![alt text][warp_test_04]

I also tested warping the image after applying the color thresholds:

![alt text][thr_warp_test_01] 
![alt text][thr_warp_test_02] 
![alt text][thr_warp_test_03] 
![alt text][thr_warp_test_04] 


###4. Identifying the Lane Line Pixels and Fit Polynomial

The relevant code can be found in the notebook "./P4.ipynb" in the section titled "Detect Lines in Warped Image"

The work is all done in the function `poly_fit()` which takes as input a binary warped image and optionally the last fit for the left and right lanes.  

If no previous fits are given, the function first uses the histogram approach to find lanes in a given horizontal section.  To find the lanes in the next higher horizontal section we slide up a window (of margin 100px) and identify the non-zero pixels inside that window.  This is continued until we reach the top of the picture.

If the function is provided with a previous fit for left and right lane, we simply look for non-zero pixels inside of a strip of with 100px around the previous fits.

After the non-zero points are found we fit a 2 order polynomial using `np.polyfit()`.

(Note: My function `poly_fit` also computes the curvature at this point, since I thought it was simpler to implement together with the fitting.)

To simplify testing this function also returns an output image showing the slide windows and the fitted polynomial.  See these four examples:

![alt text][lane_find_test_01]
![alt text][lane_find_test_02]
![alt text][lane_find_test_03]
![alt text][lane_find_test_04]

### Calculate the Radius of Curvature

The relevant code can be found in the notebook "./P4.ipynb" in the section titled "Calculate Curvature".

The function `calc_curvature` takes as input the `y` and `x` values of the non-zero points of one lane.  We then 'translate' these points from the warped pixel space to real word space in meters and fit a polynomial. Then the formula given in the lecture is applied to get the curvature for the highest `y` value, i.e. the bottom of the picture.

Next to the curvature we also return `+1` or `-1` indicating the direction of curvature (`-1` indicates a left curve).

### Illustration of full Pipeline on One Picture

The relevant code can be found in the notebook "./P4.ipynb" in the section titled "Pipeline for One Image".

To simplify storing results of previous frames I defined a class `Line` which stores the previous curve radius, the offset from center and the last left and right polynomial fits.

Steps of the pipeline:

   * Undistort the image
   * Apply color and gradient thresholds
   * Warp image
   * Fit polynomial and calculate curvature
   * Calculate offset from center of lane by comparing the midpoint of the identified lane and the midpoint of the whole picture.
   * Measure the confidence of the previous computations (we check if the lines are somewhat parallel and if the curvatures have same direction)
   * If we determined we can trust the previous computations, we update our `Line` object.
   * We plot the area enclosed by the lanes.
   * We assemble the strings which are printing the curvature estimate and the center offset on the picture.
   * Then we warp back the picture in real world space and overlay the plotted polygon and text with the original undistorted picture.

Example:

![alt text][pipeline_test_01]

---

##Pipeline (video)

Here's a [link to my video result](./out_test.mp4)

---

##Discussion

I think my approach works pretty well on the given easy video. I think the curvature I compute is a bit too string, but I don't see where I made a mistake.  

I see that the challenge videos with their frequent change of light/shadow and with their frequent curves pose some tough challenges to my approach.

To work with the challenge videos I think that first my thresholding should be more robust.  Maybe I could use more of the color channels available to deal with light/shadow differences.  I also think my fixed region of interest which gets warped is too narrow to catch the extreme curves in one of the videos.  I may also have to widen the margin in which I search for non-zero points when detecting the lanes.  Again the extreme curves might make that necessary.

Overall I could work much more on the computational performance.  I already re-use the results of previous frames and I smooth the output a bit but I think that much more of the information obtained from previous frames can be used in almost all of the steps.  E.g. why would we perform the thresholding on the whole picture if we already know where we found the lanes the last time. 


