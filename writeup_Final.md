## Final Writeup

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

[image1]: ./output_images/calibration1_undist.jpg "Undistorted"
[image2]: ./output_images/test2.jpg "Road Transformed"
[image3]: ./output_images/test2_undist_color_sobel.png "Binary Example"
[image4]: ./output_images/test2_birdview.png "Warp Example"
[image5]: ./output_images/color_fit_lines.jpg "Fit Visual"
[image6]: ./output_images/Final_information.png "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

####Please note that in this report, all the codes are within single file "Project4_Advanced_Line_Finding.ipynb". I will refer only to the cell and line numbers.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in lines 19 through 44 in the first code cell of the IPython notebook located in the main folder.  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the real world. 
Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each 
calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy 
of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the 
(x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients 
using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the 
`cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a 
thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at Cell 3
lines #30 through #131).  Here's an example of pre and post applying filters in this step. 

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I calculated my perspective transform matrix (M) and invert transform matrix (Minv) in Cell 4 line #3 to # 11. And defined the 
perspective transform function `warp_image()` in Cell 3 line #86 to line #89. The `warp_image()` function takes as 
inputs an image (`img`), and use the perspective transform matrix.  I decided to set the matrices to global variable so
 no need to duplicate the caculation. I also chose the hardcode the source and destination points in the following manner:

```python
src_corners = np.array([[580, 460], 
                        [205, 720], 
                        [1110, 720], 
                        [703, 460]])
dst_corners = np.array([[320, 0], 
                        [320, 720], 
                        [960, 720], 
                        [960, 0]])
src = np.float32(src_corners)
dst = np.float32(dst_corners)
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 580, 460      | 320, 0        | 
| 205, 720      | 320, 720      |
| 1110, 720     | 960, 720      |
| 703, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image
 and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The steps to identify lane-line pixel and fitting the positions are included in the `process_img()` function, which takes
in the source image and goes through the following steps:

1. Undistor the image (Cell 3 line #189)

2. Add a gaussian blur to prevent the sharp edge/transition (line #191 - #192)

3. Set the thresholds for color and sobel, and apply to image, then warp the image (line # 195 - #203)
    
    a. In the color filter, I used saturation from HLS image, as well as B channel from LAB color.
    
    b. For the gradient, I used x direction, and also the magnitude and direction

4. Use `find_window_centroids()` function to identify the locations of the lane lines. Details will be explained below.

5. Then apply the fitting of my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

For the `find_window_centroids()` function, several steps are taken to find the right windows for left and right lane 
lines (Cell 3 line #138 to #183)

1. Use bottom half of the image, take the histogram and use `np.argmax` function to find the peak position for the input gray scale image

2. Use the found 2 centers as the starting point, then use the 'sliding window search' method to find the followed positions

3. Here a tricky point is, for some lines it is not continuous, so it is very possible that the window is fully missed. 
In this project I juse copy the previous window position if cannot find a new position. This will cause the lines to be straight.
A better method should exist.
 
#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines #208 through #233 in my code in Cell 3 using the following approach:
1. Generate random points (x range 80) around the center found from previous step
2. Use `np.polyfit` to do 2nd order polynomial fitting, for left and right, respectively
3. Use the fitted coefficients to reconstruct the left and right curve (line #216-#219)
4. Assumed that each lane width is 370cm (according the the CHP website), calculate the position of the vehicle and return
in `off_center`(line #221). If it is negative then vehicle is moved to the left, otherwise moved to the right. Result is 
displayed in line #234-242 which also write into the output video
5. The left and right lane lines radius of curvature are calculated in line #222 and #223 


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in the function `process_image()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://youtu.be/vKWCWkf6Als)

Here's a [link to my challenge vidoe result](https://youtu.be/lPs4XtQWpsQ)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and
 how I might improve it if I were going to pursue this project further.  
 The approach I took:
 1. Build up the camera distortion matrix to undistort the image
 2. Warp the image to bird view image
 3. Use color and sobel filter to extract the lane lines
 4. 'Sliding window search' to find the line centers
 5. Use 2nd order polynomial fitting to fit the line
 6. Plot the line, and warp back to the original image
 
 I met several problems:
 1. It is very hard to find the right threshold, for different conditions, e.g., shadow, brightness change, road surface
 color change, paint etc.
 2. The gradient is also a little noisy to distinguish the shadow
 3. When the white line is dashed, for the sliding window it is hard to follow the trend. I will to do a preliminary data
 fitting to improve the window location even if there is no "white mark"

Where will the pipeline likely fail:
1. When there is a shadow or paint on the road, it may cause failure
2. Rainy road with low line contrast may fail the pipeline
3. Maybe the car run too fast
4. Changing lane

Where can be improved:
1. More robust threshold by testing more conditions
2. Machine learning?