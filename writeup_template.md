## Writeup Template

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

[image1]: ./test_images/test1.jpg "Original"
[image2]: ./output_images/output_undistorted_test1.jpg "Undistorted"
[image3]: ./output_images/output_binary_test1.jpg "Binary"
[image4]: ./output_images/output_warped_test1.jpg "Warped"
[image5]: ./output_images/output_windows_test1.jpg "Sliding Window"
[image6]: ./output_images/output_smooth_test1.jpg "Fit Smooth Curve"
[image7]: ./output_images/output_warpedmasked_test1.jpg "Warped with mask"
[image8]: ./output_images/output_final_test1.jpg "Output"
[image9]: ./camera_cal/calibration3.jpg "Chess Calibration Image"
[image10]: ./output_images/undistorted_chess.jpg "Undistorted Chess Calibration Image"
[video1]: ./output_video/Project_Video_Gaby_GRLS.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./P4.ipynb"

I started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 
Before - Original Image:
![alt text][image9]

After - Undistorted Image
![alt text][image10]

At the end of this section I save my `mtx` and `dist` data from the calibration in a pickled file called `calibration\_pickle.p` to be used later on.

### Pipeline

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, here is a before and after of the same test image:

Original:
![alt text][image1]
Undistorted:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines #23 through #102 in the third code cell of my jupyter notebook).

I tested thresholding together the binary values of different color channels and color spaces with the gradient in the x direction of the image. The combination of the gradient in the x direction with the S binary value, R binary value, and V binary value yielded an okay result for the most part. When the pipeline was used in the project video the tracking worked fine for almost the entire duration of the video except when driving on the light concrete section around second 23 and again around second 39, where the yellow line becomes very light and hard to see and the program starts to detects shadows by the median as lane lines causing the tracking to jump to such detections. An example of a video with this 'unsuccessful' thresholding can be found [HERE](./output_video/Project_Video_Gaby_SRV.mp4)

In attempts to fix this, many other thresholding values for different color channels and color spaces were tried, landing to the successful combination that can be found in line #76 of the third code cell of my jupyter notebook, which shows a thresholding combination of the gradient in the x direction, binary G value and binary R value, and binary L value and binary S value. the R and G where chosen because the challenge was mainly in detecting the color yellow, which is a combination of the green and red color channels. The L (lightness) and S (saturation) where used because another challenge was the change in lightness of the concrete in comparison with the asphalt and the saturation needed to make the color yellow stand out. A successful output video when using this thresholding can be found towards the end of this write up.

Here's an example of the output of the binary thresholding in a test image:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

For my perspective transform I simply used different cv2 functions and different chosen and calculated source (src) and destination (dst) points. Refer to lines #97 through #100 in the third code cell of my jupyter notebook for the implementation.

The two cv2 functions used were the `cv2.getPerspectiveTransform` to get the Matrix using the `src` and `dst` points. Then again in reverse order to get the inverse matrix. And the `cv2.warpPerspective` to get the warped image using the previously calculated matrix.

Next, is a section of the code where the src and dst points were calculated:

```python
w,h = 1280,720
x,y = 0.5*w, 0.8*h
src = np.float32([[200./1280*w,720./720*h],
                [453./1280*w,547./720*h],
                [835./1280*w,547./720*h],
                [1100./1280*w,720./720*h]])
dst = np.float32([[(w-x)/2.,h],
                [(w-x)/2.,0.82*h],
                [(w+x)/2.,0.82*h],
                [(w+x)/2.,h]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 200, 720      | 320, 720      | 
| 453, 547      | 320, 590      |
| 835, 547      | 960, 590      |
| 1100, 720     | 960, 720      |

The warped image result in a test image is the following:

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To identify the lane line pixels and fit with a polynomial I used the histogram and sliding window method. where I plotted a histogram of the warped images and found its peaks. Then created a total of 9 windows to scan the image from bottom to top around the peaks of each side (left and right lane lines) with a set margin and minimum pixel number for each window. As the program scanned, the values of x and y where stored and then filtered to only keep the good 'usable' values. These values where concatenated and used to fit the second degree polynomial. Refer to lines #104 through #215 in the third code cell of my jupyter notebook.

I output three different images for these section. I believe all three represent very well what's happenning to each image. 
First the image is scanned using 9 leyers of windows from bottom to top for each the left and the right lane line:
![alt text][image5]

Then a smoother curve is fitted to the lane lines:
![alt text][image6]

Lastly, a the `cv2.fillPoly` is used to fill a polynomial with a green mask created using the found lane line image coordinates:
![alt text][image7]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The calculation of the radius of curvature and position of the car with respect to the center of the lane line was calculated from line #218 through line #243 of the third code cell in my jupyter notebook. There, the formula to calculate the radius of curvature was used (refer to lecture 35 "Measuring Curvature" of the Advance Lane Finding lesson). Refer to line #221 for the calculation of the left curvature using the values obtained from the left half of the image. Refer to line #222 for the calculation of the right curvature using the values obtained from the right half of the image.

To calculate the position of the vehicle with respect to the center of the lane lines, lines #239 through #243 were used. Where the program calculates the center using the left x values and right x values of the lane lines and dividing by 2 to find the center and then makes a comparison so that if the center minus half the width of the image is greater than 0 then that means the car is to the left of the center and if the number is less than or equal to 0 then the car is to the right of the center of the lane lines.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines #262 through #269 in the third code cell of my jupyter notebook, where I used the same `cv2.warpPerspective` function but using the inverse matrix `Minv` to reverse the image back from the warped state to an unwarped state (or back to normal). 

Here is an example of my result on a test image:

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_video/Project_Video_Gaby_GRLS.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The project went relatively smooth. The one bump on the road was definitely the change in ground color when the video goes from the dark asphalt to the light concrete back to the dark asphalt as mentioned earlier in this write up. Playing around with the color and channel spaces was good enough in this case and for this implementation, but a better aproach could be to make the pipeline remember the last successful left and right points found if/when the next point on the road differs from the previous by a large margin. Meaning that the program would have to remember the last successfull state until finding more successul, "within range", points on the road. In general a very interesting project that teaches a great deal about image processing to achieve a successful lane line tracking.