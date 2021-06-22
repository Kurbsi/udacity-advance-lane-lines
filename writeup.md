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

[image1]: ./output_images/undistorted_output.jpg "Undistorted"
[image2]: ./output_images/straight_lines1_undistorted.jpg "Road Transformed"
[image3]: ./output_images/binary_thresholded.jpg "Binary Example"
[image4]: ./output_images/warped_straight_lines.jpg "Warp Example"
[image5]: ./output_images/color_fit_lines.jpg "Fit Visual"
[image6]: ./output_images/example_output.jpg "Output"
[image7]: ./output_images/histogram.jpg "Histogram of activate pixels"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.

---

### Writeup / README

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is in "./lane_lines.ipynb". The function cal_undistort uses internally the OpenCV function `cv2.calibrateCamera()`. This function needs as argument mainly the object points, the image points and the size of the image. We ignore the remaining arguments and set them to None.

This means we need to prepare the object points and the image points.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image. The important functions are `sobel_threshold(img, sx_threshold=(20,100))`, `direction_threshold(img, dir_thresh=(np.pi/6, np.pi/2))` and `color_channel_threshold(img, l_thresh = (120, 255), s_thresh=(100, 255))`. In addition we only activated pixels with a R and G channel above 50 to have a higher chance of detecting yellow lines properly. A threshold of 50 was found by trial-and-error.

Here's an example of my output for this step.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

To finally get the respective road lines, we need to first transform the road onto the image plane. To do so, we need to calculate the transformation matrix.

In the code this is done in the function `get_transformation_matrix(img)`. To retrieve the transformation matrix again OpenCV provides the build in function `getPerspectiveTransform(src, dst)`. The Source points `src` where found from one original image as 

| Source      |
|:-----------:|
|[580,460]    |
|[700,460]    |
|[1045,680]   |
|[275,670]    |

To receive the respective destination points `dst`, I assumed that the road lines from the original image are parallel. Therefore the destination points have to also be parallel on the resulting image. I choose to not put the destination points right on the edges of the resulting image but to offset them by 150 px from left and right to get a better impression.

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Next step is to detect the actual lines from the binary warped image. To do so we create a histogram of activated pixel. I choose to only create a histogram of the lower part of the image, since there the lines tend to be mostly vertical.

![alt text][image7]

As can be seen both lines, especially the left one are clearly visible. This can be used in the function `def find_lane_pixels(binary_warped)`. 

I extract the peaks in the histogram, which will now be the starting point of a sliding window approach to detect the lines in the image. Therefore we extract all non-zero pixel in the image and divide the vertical slice of the image in between 9 windows. If the non-zero pixel in the respective window surpass the threshold of 50 pixel we expect a line segment to be found and update the current base so that we can continue from there with the new window. Finally we get the actual pixels 

Now that we found all the line pixel we can simply fit a second order polynomial through all of the so that we have a mathematical representation of our lines.

The following image shows the windows which where found by the sliding window approach for the left line, colored in red, and the right line, colored in blue. Also in yellow a representation of the calculated line is plotted onto the image.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

One additional step was to calculate the curvature of the lane as well as the position of the vehicle w.r.t. the lane. This is done in the functions `calculate_curvature(left_plot, left_fit, right_fit)` and `calculate_lane_offset(image, left_fit, right_fit)`

To calculate the curvature, I use the highest y value (which corresponds to the lane found closest to the car). In addition I need to know how much one pixel in image space corresponds to distance in the real world. This is done as follows:
```
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension
```

To calculate the offset from the center line I first need to know where (in pixel space) the centerline is. Therefore we need to evaluate the two line bases for the left and right line. This can be easily derived again from the previously calculated fit as the parameters of a second order polynomial function. Evaluating this function for the image height transformed to real world distance, returns the real world base for the left and right line. The average of both finally evaluates to the centerline (`lane_mid`).

For calculating the car's position I assume that the camera is located at the center of the car, so that the center of the image refers to the center of the car `car_position`. Substracting now the `car_position` from `lane_mid` returns the distance of the car from the centerline, where a negative results indicates that the car is right of the lane centerline and vice versa.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

In the final step we transform the found lane area back onto the original image. Therefore we first fill the found area being the lane area on the warped image and then use the inverse transformation matrix to get back the original image with the 

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Although some energy was put into tweaking the threshold and different filters the pipeline still fails for more uneven situation. This becomes aware in the two more challenging videos, where more difficult scenarios are driven. Especially when the contrast between the lines get worse, when lines are more uneven or especially when changing lightning conditions with shadow and bright sun light comes into play the pipeline quickly falls apart. An even further filtering of the oncoming image would here be neccessary to handle these situations properly.

Also some time could be invested in making the whole pipeline more stable against outliers or situations where no lines are found. Currently we simply assume a line in those scenarios.

Lastly, the performance of the pipeline could obvisously be improved. Currently processing the 50s project_video takes around 2.5 min, which is far of from being usable in any real world in car situation.
