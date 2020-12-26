# **Advanced Lane Finding Project**

---

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

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/pipeline_output.png" width="500" />
</div>

##### [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

##### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

### Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the `calibrate_camera()` function "AdvancedLaneFinding.ipynb".

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  
I applied this distortion correction to the test image using the `cv2.undistort()`, computed by the undistort() function and obtained this result: 

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/undistorted_calibration_output.png" width="500" />
</div>

## Pipeline (single images)

### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/undistorted_output.png" width="500" />
</div>

### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image, the steps can be found in the `hls_select()` function.

In details from the original image I extract the following:
* From the RGB Image I extract the Red Channel and applying a binary threshold (r_thresh) I extract only the needed information.
* From the HLS Image I extract the H, L and S Channels and for each of them:
    1. H Channel: applying a binary threshold (h_thresh)
    2. L Channel: computing the Sobel along the x axis and applying a binary threshold (l_thresh)
    3. S Channel: applying a binary threshold (s_thresh)

In the end I compute the output binary image by applying the OR operation between the single Binary threshold Red, H, L and S.

color_binary[(l_binary_output == 1) | (r_binary_output == 1) | ((s_binary_output == 1) | (hue_binary_output == 1))] = 1

Here's an example of my output for this step.

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/binary_output.png" width="500" />
</div>

The binary threshold used in the Project Videos are the following:
* Project Video: hls_select(img, s_thresh=(100, 255), l_thresh=(10, 100), r_thresh=(200, 255), h_thresh=(18,25), debug=0)
* Challenge Video: hls_select(img, s_thresh=(50, 255), l_thresh=(50, 100), r_thresh=(180, 255), h_thresh=(20,25), debug=0)

### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code that generates my perspective transform is contained in the `undistort_and_unwarp()` function of "AdvancedLaneFinding.ipynb", it includes a function called `cv2.warpPerspective()`.  
The `undistort_and_unwarp()` function takes as inputs the following parameter:
* `img`: the input image
* `nx`, `ny`: number of points along the axis
* `mtx`, `dist`:  camera calibration parameter
* `src`: the source matrix
* `offset`: offset used to compute the `dst` destination matrix by 
            dst = np.float32([
                [offset, 0],                       # top-left corner
                [img_size[0]-offset, 0],           # top-right corner
                [offset, img_size[1]],             # bottom-left corner
                [img_size[0]-offset, img_size[1]]  # bottom-right corner
            ])

I chose to hardcode the source points in the following manner:
* Project Video:
                | Source        | 
                |:-------------:| 
                | 590, 440      | 
                | 690, 440      |
                | 180, 680      |
                | 1100, 680     |
* Challenge Video:
                | Source        | 
                |:-------------:| 
                | 590, 460      | 
                | 690, 460      |
                | 180, 680      |
                | 1100, 680     |


Here's an example of my output for this step.

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/wrapped_output.png" width="500" />
</div>

### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Using the perspective trasformed binary images computed in the previous step I can now plot an histogram on where the binary activation occurs across the x-axis. Hight values in the histogram identify where the lane lines are more likely to be.
From the histogram I then divide the image based on the midpoint between the two highest histogram values and apply the sliding windows technique to search for the lane lines across the entire image starting from the bottom hight value of the histogram.
All of this can be found in the `fit_polynomial_first()` and `find_lane_pixels_first()` functions of "AdvancedLaneFinding.ipynb".

Once I've identifyed the lane lines of the first image i can switch to anothor tipe of search. The idea is that in the next image the lane line are likely to be in the same area of the lane lines identified in the previous image. 
So in the `fit_polynomial_prior()` function of "AdvancedLaneFinding.ipynb" we are searching the new line lines using the previous lane lines +/- a margin as reference.

Here's an example of my output for this step.

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/fit_polynomial_first_output.png" width="500" />
</div>

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/fit_polynomial_prior_output.png" width="500" />
</div>

### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I compute the curvature and the distance from center in the `get_curvature_and_center_delta()` function of "AdvancedLaneFinding.ipynb".
In detail I call the `get_curvature_in_meter()` function of the Line Class that convert the lane fit in meters and then compute the curvature of the lane line. 
At last the average between the left and right line curvatures is computed and returned.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Ihis step is implemented in the `invert_transform()` function of "AdvancedLaneFinding.ipynb".
Here's an example of my output for this step.

<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/pipeline_output.png" width="500" />
</div>

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

* Project Video:
<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/project_video_gif.gif" width="500" />
</div>

* Challenge Video:
<div align="center">
    <img src="https://github.com/Ventu012/P2_AdvancedLaneLines/blob/main/output_images/challenge_video_gif.gif" width="500" />
</div>

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The main issues I faced during the implementation of this project are the following: horizon detection and binary threshold parameters.
Basically this issues presented when I obtain a good result on the Project Video and decided to apply it also on the Challenge Video.
Not only the region of interest was wrong but also the binary threshold parameters used previously were not correct for the Challeng and I had to change both values manually.

To make it more robust I could implement some sort of horizon detection using sobel-y and from there derive the region of interest, also for the binary threshold parameters we could try to find the best set of parameters to cover most of the situation found on the road.
This could be done by dividing the original image in smaller image and, after computing the overall luminosity of the single smoller image, apply a specific set of binary threshold parameters to the smoller image. 

Both of this suggestion could be usefull in the harder_challenge_video. In this video not only the horizion changes throught the entire video but also due to the frequent changes from sunny areas to shadow areas it is difficult to extract all the information using a single set of binary threshold parameters.



