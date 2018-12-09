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

[image1]: ./output_images/calibrate.png "Undistorted"
[image2]: ./output_images/undistort.png "Road Transformed"
[image3]: ./output_images/binary_image.png "Binary Example"
[image4]: ./output_images/birds_eye.png "Warp Example"
[image5]: ./output_images/polyfit.png "Fit Visual"
[image6]: ./output_images/result.png "Output"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 3rd code cell of the IPython notebook located in "./AdvanceLaneFinding.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained the result below. The calibration paramters, mtx and dst, are saved in the global workspace for use in the `process_image()` function.

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

I applied the distortion correction using the `cv2.undistort()` function and the calibration parameters computed from the camera calibration step.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a color binary image in cell 5. I created a binary image so it I could see the influence of each threshold method. I applied an x gradient threshold on lightness and a magnitude threshold on saturation. I found that the saturation threshold also picked up shadows. In order to filter out shadows from the saturation binary, I applied a threshold on hue centered on yellow +-50 deg. This created a mask that excluded shadows. I applied this mask to the saturation binary to get a saturation binary that excluded shadows. Finally, the result is also masked by a trapezoid designating the region of interest.

Here's an example of my output for this step. I subsequently converted it into a single-channel binary in `process_image()` prior to perspective transform.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I created a function called create_birds_eye_view in cell 7. This simple function applies `cv2.warpPerspective()` to the image using the perspective matrix. The perspective matrix is computed in cell 6. I tuned the scale factors for the source and destination points until the lane lines in the transformed image were parallel. The resulting perspective matrix was saved as a global variable for use in the `process_image()` function. An example image that demonstrates parallel lines is shown below.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Ths functions for identifying lane line pixels and fitting their positions are in cell 8. In the first pass, the lane-line pixels are identified using a step-wise windowing method. The initial centroids of the left and right windows at the bottom of the image are based on the two peaks of the histogram of the lower half of the image. The next windows one level up from the previous windows are centered on the mean of the previous level. This is repeated until the top of the screen. Then a polynomial fit is applied to the points in the left and right window sets.

Once a polynomial fit for the left and right lanes is computed, subsequent windows are directly computed around the polynomial fit lines from the previous video frame. This is much faster than the step-wise windowing method. The polynomial fits from the previous frame are saved as global variables for use in the next call to `process_image()`. Here is an example of the lane-line pixels and polynomial fit:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature and lateral position calculations are in cell 9.

In addition to a polynomial fit in pixel space, a polynomial fit in units of meters was also computed by scaling the pixels to meters prior to computing the fit. The radius of curvature is computed at the bottom of the image directly from the coefficients of the polynomial. The bottom of the image is closest to the vehicle and therefore most relevant. Noise in the transformed image tends to result in a wide "smeared" point cloud which tends to lower the perceived curvture. In order to make the system robust to this effect, the software reports the lower of the left and right curve radii.

Likewise, the lateral position is computed based on the mean position of the lines at the bottom of the image with respect to the center of the image.

The curve radius and lateral position are low pass filtered to reduce noise. The time constant for the radius is higher than the time constant for the lateral position because the true lateral position changes at a faster rate than the lane curvature.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The complete pipeline is assembled in `process_image()` in cell 12. Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The binary image generator can be further improved by identifying corner cases and updating the threshold methods.

A validator can be implemented to identify and throw out unreasonable solutions. When an unreasonable solution is thrown out the previous solution is reported until a valid solution is returned.

The current implementation only applies the lane-line pixel bootstrap detector in the first pass. This can be modified to do a bootstrap every time an unreasonable solution is returned.

The current implementation only low pass filters the radius and lateral position. It would be more robust to low pass filter the lane lines themselves.
