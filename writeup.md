# Advanced Lane Finding Project
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

[image1a]: ./camera_cal/calibration1.jpg "Original"
[image1b]: ./examples/Step2_undistorted_calibration1.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/Step3_thresholded_test1.png "Binary Example"
[image4a]: ./examples/Step4_warpedIn_straight_lines1.png "Warp Source"
[image4b]: ./examples/Step4_warpedOut_straight_lines1.png "Warp Result"
[image5]: ./examples/Step5_lines_warped_thresholded_test1.png "Fit Visual"
[image6]: ./examples/Step67_result_undistorted_test1.png "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

You're reading it!

### Camera Calibration

The code for this step is contained in section 1. and 2. of the IPython notebook located in "./advLane.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

_Original:_![alt text][image1a]

_Undistorted:_ ![alt text][image1b]

### Pipeline (single images)


#### 1. Distortion-corrected image
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Thresholded binary image

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in section 3. of "./advLane.ipynb", lines 18 through 35).  Here's an example of my output for this step.

![alt text][image3]

#### 3.Perspective transform

The code for my perspective transform is in section 4. of "./advLane.ipyb" IPython notebook. The code takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 58, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 5), img_size[1]],
    [(img_size[0] * 5 / 6) + 45, img_size[1]],
    [(img_size[0] / 2 + 63), img_size[1] / 2 + 100]])

dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points (x,y are rounded in this table):

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 582, 460      | 320, 0        | 
| 208, 720      | 320, 720      |
| 1112, 720     | 960, 720      |
| 703, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

_Source image_
![alt text][image4a]

_Result image_
![alt text][image4b]

#### 4. Finding lane-line pixels and fitting their positions with a polynomial

We use sliding windows on sub-divided image to find relevant lane-line pixels, that will be used for polynomial fitting.

First we take a histogram of the bottom half of the picture, and find the x-position for maximum value for left and right half of the image. This gives two starting points, around which we will search for non-zero pixels (search-window is a rectangle). We re-center the next windows based on mean positions of the current windows. We then repeat the search in the next windows, which are previous windows shifted vertically (and optionally re-centered with latest lane finding).

This step will give a set of left and right points that will characterize the left and right lanes. We then fit a second degree polynomial to summarize left and right lanes. Image below shows those steps. The corresponding code is in section 5. of "./advLane.ipyn" IPython notebook.

![alt text][image5]

#### 5. Radius of curvature of the lane and position of the vehicle with respect to center.
We can extract the curvature from the polynomial coefficients previously found. The polynomial must be adjusted to represent (x,y) in meter instead of pixel. For this, we re-generate (x,y) points and adjust them with (xm_per_pix, ym_per_pix) as a scaling factor. We then can re-fit those points to find the polynomial coefficients in meter, that will be used to compute the curvature (from curvature equation).

For the lane position with respect to center of the vehicule, we simply compute the center of our lane finding, and compare it to the center of the picture. This will give the offset of the vehicule w.r.t lane center (we assume the camera is on the center of the vehicule, so being in the middle of the lane means being in the center of the image). Here again, we need to convert pixel distance to meter to get the correct information.

This is summarized in `get_curvature_real` and `get_offset` functions from section 6 of "./advLane.ipyn" IPython notebook.

#### 6. Example image of lane finding plotted back down onto original image.

I implemented this step in section 7. of of "./advLane.ipyn" IPython notebook. We use `Minv` to unwarp the lines onto the original image. We also add the curvature and offset information on the original image. Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

Here we build the pipeline described in previous sections. The pipeline had issues when the color of the road changes dramatically. To deal with abrupt changes, we save an history of lanes so that we can average the lanes over the last n images (using 60 here). We also can set a search window from previous lanes, which means we have a good intuition where to search already. Finally, we try to discard problematic image by checking if lanes are not parallel. This improves the lane finding pipeline overall.

Here's a link to [project_video_output](./examples/project_video_output.mp4)

It gives somewhat acceptable results on the challenge video: [challenge_video_output](./examples/challenge_video_output.mp4)

It fails to find lanes on the harder challenge video: [harder_challenge_video_output](./examples/harder_challenge_video_output.mp4)

---

### Discussion

The history helped improving results a lot, but it also makes the lane detection not very reactive, so that when the curvature is strong, the pipeline lags too much which is problematic (we need to react fast during catastrophic situation).

One improvement would be to weight the history based on lane finding confidence and maybe a decaying factor.

Another improvement would be to adjust the perspective transform window. Typically, is the curvature is strong, we might want to look for lanes in the bottom quarter of the image instead of the bottom half (in another word, having several perspective transform matrices instead of only one).

Finally, sanity check and fall back mechanisms (when no lanes are found) could be improved.
  
