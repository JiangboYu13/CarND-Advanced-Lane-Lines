## CarND-Advanced-Lane-Lines Writeup 

### This writeup briefly summary the work done for advanced  lane detecton project

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

[corner_board]: ./output_images/draw_on_board/DRAW_calibration3.jpg 
[undist_corner_board]: ./output_images/undist/calibration3.jpg 
[undist_img1]: ./output_images/undist/test4.jpg 
[undist_img2]: ./output_images/undist/test5.jpg 
[filtered_img1]: ./output_images/filtered/test4.jpg 
[filtered_img2]: ./output_images/filtered/test5.jpg 
[warped_img1]: ./output_images/warped/test4.jpg 
[warped_img2]: ./output_images/warped/test5.jpg 
[hist1]: ./output_images/histogram/test4.jpg 
[hist2]: ./output_images/histogram/test5.jpg 
[polyfit1]: ./output_images/polyfit/test4.jpg 
[polyfit2]: ./output_images/polyfit/test5.jpg 
[result1]: ./output_images/result/test4.jpg 
[result2]: ./output_images/result/test5.jpg 
[result_video]: ./test_videos_output/project_video.mp4 

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---



### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is defined as function named `calCamera(...)` in IPython notebook located in `./pipeline_code.ipynb`

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `obj_pnt` is just a replicated array of coordinates, and `obj_pnts` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `img_pnts` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. Here is an example of corners found on board:

![corner board][corner_board]

I then used the output `obj_pnts` and `img_pnts` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![undistorted cornerboard][undist_corner_board]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![undistorted image 1][undist_img1]
![undistorted image 2][undist_img2]
#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (defined as function  `hls_select(...)` in `pipeline_code.ipynb`). 
```python
     binary_output[((sat>30)&(lig>30)&(abs_sobel_thresh(img,orient='x' sobel_kernel=5, thresh=(15, 255))==1))|(lig>200)|((lig>140)&(sat>100))] = 1
```

Here's an example of my output for this step.  (note: this is not actually from one of the test images) 

![filtered image 1][filtered_img1]
![filtered image 2][filtered_img1]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper(...)`, which appears in `pipeline_code.ipynb`  The `warper(...)` function takes as inputs an image (`img`), In the warper function, another function `warpTrsf()` which will return the  Perspective Transform Matrix. I chose the hardcode the source and destination points in the following manner:

```python
 src = np.array([[245, 692],[1058, 692],[702, 460],[582, 460]], np.float32)
    dst = np.array([[1/4, 1,], [3/4, 1], [3/4,1/10], [1/4, 1/10]],np.float32)
    dst[:,0] = dst[:,0]*1280
    dst[:,1] = dst[:,1]*720
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 245, 692      | 320, 720      | 
| 1058, 692     | 960, 720      |
| 702, 460      | 960, 72       |
| 582, 460      | 320, 72       |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![warped image1][warped_img1]
![warped image2][warped_img2]
#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?
First I need to check if previous knowledge of polynomial line is avaliable. If avaliable, it will search in the neighbor of previous polynomial(defined in  function `search_around_poly(...)` ). Otherwise, it will search from scratch(defined in  function `search_blind(...)` ).

To search blind, I first use histogram to find out the most likely `x` coordinates for left lane and right lane as the start x-centre point. Then I search from bottom to top trying to find all pixels with none-zero value. The x-centre point will be re-centreed whenever there are more than 40 pixels found. 

To search around known polynomial, I use the x coordinate on the polynomial as x-centre point and search from bottom to top.


Then I fit my lane lines with a 2nd order polynomial kinda like this:

![polyfit1][polyfit1]
![polyfit2][polyfit2]
#### 4. The line sanity checking and smoothinng
The sanity check process is defined in function `sanityCheck` in `pipeline_code.ipynb`.
It first choose more than 10 points equually seperated in y-direction from left polyfit and obtain the pamameters for lines perpendicular to each point respectively. Then find the intersection points between right polyfit and the perpendicular lines. Find the distance between points left polyfit and right polyfit in pair and calculate the standard deviation. Polyfit lines with standard deviation less than 50 are regarded reasonable. 

The smoother is defined as a class `Lines` in `pipeline_code.ipynb`. 
This class has following attribute: 
- _line_queue
- _queue_size
- _bad_frame_count
When add new polifit lines, it will first perform sanity checking. 

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in function `measure_curvature_real(...)` in `pipeline_code.ipynb`.

curvature is obtained using fomula:
```math
    R_curve = ((1+(2Ay+B)^2)^(3/2))/|2A|
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in function `polyUnwarp(...)` and `laneOverlap(...)`in `pipeline_code.ipynb`.  Here is an example of my result on a test image:

![result1][result1]
![result2][result2]
---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).
Here's a [link to my video result](./test_videos_output/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
