# CarND Project 4: Advanced Lane Lines

[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)

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


[calibration1]: ./camera_cal/calibration2.jpg "Distorted"
[calibration2]: ./output_images/camera_chessboard/chessboard_11.jpg "Chessboard"
[calibration3]: ./output_images/camera_undistorted/undistorted_11.jpg "Undistorted"

[original1]: ./test_images/test1.jpg "Distorted"
[undistorted1]: ./output_images/test_images_undistorted/undistorted_02.jpg "Undistorted"
[threshold1]: ./output_images/threshold/binary_02.jpg "Thresholded"
[warped1]: ./output_images/warped/warped_02.jpg "Thresholded"

[perspective]: ./output_images/perspective_transform.jpg "Perspective transform src and dst points"

[sliding_window]: ./output_images/draw/draw_06.jpg "Detected lane pixels"
[hud]: ./output_images/hud/hud_06.jpg "Lane drawn onto original image"


### [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

## Camera Calibration

### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

When taking images with a camera, the lens mapping the 3D world to a 2D image causes distortions. Distortions interfere with geometric calculations based on images. Therefore it is necessary to determine the lens' distortion in order to be able to neutralize it. To do that, I use the provided chessboard images taken with the camera used for the images and videos in this project. My implementation of the chessboard detection and camera calibration is located in [helpers/calibration.py](helpers/calibration.py). It uses `cv2.findChessboardCorners(...)` to find the coordinates of chessboard intersections and calculateds the transformation matrix to undistort the image via `cv2.calibrateCamera(...)`. The resulting matrix is then used to apply `cv2.undistort(...)` and undistort an image.

| Original out of camera image | Detected chessboard | Undistorted image |
|:---:|:---:|:---:|
| ![Original distorted image][calibration1] | ![Original image with chessboard][calibration2] | ![Undistorted image][calibration3] |


## Pipeline (single images)

### 1. Provide an example of a distortion-corrected image.

Here's an example of the above distortion correction being applied to a road image from the video:

| Original image | Undistorted image |
|:---:|:---:|
| ![Original image][original1] | ![Undistorted image][undistorted1] |


### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I tried out several color thresholds and Sobel filters as well as combinations of both. All implementations I tried can be found in [helpers/color.py, line 52 and following](helpers/color.py#L52). It seemed that pure color thresholds yield more reliable results as edge detection often detects edges irrelevant to lane finding and therefore confuses the algorithm during further processing. I chose to use the color spaces HLS, HSV and Lab for color thresholding. The final implementation is called via [`ColorThreshold.threshold(...)`](helpers/color.py#L57), which in turn ends up calling [`ColorThreshold._do_thresholding(...)`](helpers/color.py#L102).

| Undistorted image | Thresholded image |
|:---:|:---:|
| ![Undistorted image][undistorted1] | ![Thresholded image][threshold1] |


### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

In order to determine the lane curvature, it is easier to use a birds eye view. To simulate that, we can use perspective transformation. I took one of the straight lane images from the test images and picked four source points below.

[`helpers/warp.py, line 9-21`](helpers/warp.py#L9)
```python
    @property
    def source_points(self):
        return np.array([
            (230, 700), (1075, 700), (693, 455), (588, 455)
        ], np.float32)

    @property
    def destination_points(self):
        offset = 200
        x1, x2 = 640 - offset, 640 + offset # 440, 840
        return np.array([
            (x1, 720), (x2, 720), (x2, 0), (x1, 0)
        ], np.float32)
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 230, 700      | 440, 720      | 
| 1075, 700     | 840, 720      |
| 693, 455      | 840, 0        |
| 588, 455      | 440, 0        |

I verified that my perspective transform was working as expected by drawing the source and destination points onto both straight lane line test images and its warped counterpart to verify that the lines appear parallel in the warped image.

![Source and destination points][perspective]


### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

My implementation of the lane finding is inside the class `LaneSearch` of the module [`helpers/lane.py`](helpers/lane.py#L9) and in turn uses the custom data structure `Line` from [`helpers/line.py`](helpers/line.py#L6). The general approach of the lane detection is to construct a histogram of the lower part of the image. I chose to use the bottom quarter, rather than the bottom half. The histogram will have peaks on those x-coordinates where a lot of white pixels were detected. Those are the lane lines. Starting from those two x coordinates we start looking for lane lines with the sliding window approach, which was introduced in our lesson. The pixels identified as lane line pixels by the sliding window function ([`LaneSearch._sliding_window`](helpers/lane.py#L55)) are then used to fit a second degree polynomial ([`np.polyfit(...)`](helpers/lane.py#L130)), which results in a quadratic function, i.e. `f(y) = a * y^2 + b * y + c`. The function is a function of `y` instead of `x` because we also have perfectly straight and vertical lane lines, which would result in very large coefficients. In order to avoid that, the fitted function is calculated as function of `y`.

![Sliding window][sliding_window]


### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The curvature is calculated inside the sub-function [`curvature(...)`](helpers/lane.py#L265) as learned in the lesson. I calculate the radius at the bottom of the image, near the car. The curvature of the left and right lane lines are calculated separately. The average of both is then displayed.

The position of the vehicle compared to the lane center is also calculated ([`off_center(...)`](helpers/lane.py#L262)) and displayed.

For both metrics it is necessary to first convert the pixel measures to real-world measures in meters, which is done in [`LaneSearch.draw_lane, line 259-260`](helpers/lane.py#L259).


### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Finally, the function [`LaneSearch.draw_lane`](helpers/lane.py#L244) takes care of display all the acquired information back onto the original image as shown below.

![Resulting image][hud]


---

## Pipeline (video)

### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

[My video result](./out.mp4)

[My challenge video result](./out_challenge.mp4)

---

## Discussion

### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I spent considerable amount of time trying to optimize the thresholding and every other step of the process independently. Experience taught me that it's better to get a working pipeline first and then start tweaking parameters, etc.

I still think that my thresholding can be improved, especially for different lighting conditions. Often tweaking the thresholds for one video resulted in worsened results in another video. In the end I focussed on optimizing for the first video.

As for the polynomial lane line fitting, it is not realistic to assume that a lane only bends one way. Especially with countryside roads, like in the harder challenge video, lanes can have `S` line shapes, which would be better represented with a 3rd order polynomial.

Furthermore, very strong bends like in the harder challenge video, are difficult to find using the sliding windows, because the perspective transformed lines are partially horizontal. In the same harder challenge video it is also visible that in strong bends one line can end up completely outside the camera's view. Using two or three cameras could help mitigate this issue as well as potentially the next one.

Another problem occurs when lane lines are partially covered by other vehicles, which significantly reduces the ability to detect those lines.

Performance-wise, processing the videos took longer than their runtime. This means that the current algorithm can't be applied in real-time on my laptop.

As we had a deep neural network drive around a simulator car racing track, I could imagine that a DNN approach to lane finding could also work well.
