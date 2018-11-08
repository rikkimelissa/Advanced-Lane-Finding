**Advanced Lane Finding Project**

The goal of this project is to write a software pipeline to identify the lane boundaries in a video from a front-facing camera on a car. 

[//]: # (Image References)

[image1]: ./output_images/chessboard1.png "Finding chessboard corners"
[image2]: ./output_images/chessboard2.png "Finding chessboard corners"
[image3]: ./output_images/chessboard1dist.png "Raw camera image"
[image4]: ./output_images/chessboard1undist.png "Undistorted camera image"
[image5]: ./output_images/fill_poly.png "Using fitted lines to draw a polygon"
[image6]: ./output_images/perShiftpoly.png "Polygon after perspective shift"
[image7]: ./output_images/perTransAft.png "After perspective shift"
[image8]: ./output_images/perTransb4.png "Before perspective shift"
[image9]: ./output_images/rectangles.png "Using sliding window approach"
[image10]: ./output_images/thresholding.png "Thresholding image with color and edges"
[image11]: ./output_images/left_fit.png "Fitting left lane from previous"
[image12]: ./output_images/right_fit.png "Fitting right lane from previous"

### Here I will describe the steps I took to reach this goal.

---


### Camera Calibration


The code for this step is contained in the first code cell of the IPython notebook located in "LaneFinding.ipynb". Most of this code comes from the lessons preceding this project.  

As we did in the practice sections before this project, I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

Here are two examples of finding chessboard corners:
![alt text][image1]
![alt text][image2]

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result, shown before and after distortion: 

![alt text][image3]
![alt text][image4]


### Perspective shift

The code for this step is contained in the second cell of the notebook. I hardcoded source and destination points in an example undistorted straight line image by selecting near and far points on the right and left lane lines, and shifting them into a rectangle taking up most of the image. I use these points to calculate a perspective shift matrix and inverse matrix, which I then use in all future perspective shifts.

Here is an example of an image before and after the perspective shift:

![alt text][image8]
![alt text][image7]


### Pipeline 

The rest of the code for my image pipeline is in the third cell of the notebook. I use the Line class to hold information between successive image processing. This class contains functions for distortion, thresholding, fitting lines to curves, drawing a polygon and performing a perspective shift, and calculating curvature. The main function is called process_image and runs through the image pipeline. The source for a lot of this code is the information provided in the lessons.

#### 1. Distortion 

The first step is to undistort the image (lines 25-27). This function was described above with example images.

#### 2. Thresholding 

As we did in the lessons, I used a combination of color (the saturation channel in the HLS spectrum) and gradient (the absolute Sobel derivative in the x direction) thresholds to generate a binary image (lines 30-53).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image10]

#### 3. Perspective transform

Next, I shifted the image to a top-down perspective using the precalculated matrices (line 204). This function was described above with example images.

#### 4. Identifying lane pixels and line fitting

Next, I used one of two lane fitting functions to identify pixels and fit a second degree curve. Both lane fitting functions return the average residual value of the fitted line, the value of which determines whether or not the fit will be used.

The first function (lines 56-126) fits a line from an image with no prior information. It performs a histogram on the bottom half of a binary image, and uses the highest values on the left and right side as an initial guess for the bottom of the lane line. It steps up the image in a series of sliding windows, using the horizontal mean of the current window's positive pixels to determine the center for the next window (if a certain number of positive pixels have been detected). Once all pixels in the sliding windows have been identified, a second degree curve is fit to these pixels. An example of this sliding window approach and the resulting curve fit:

![alt text][image9]

The second function (lines 129-153) fits a line from an image using the previous image's curve fits as a starting guess. Window boundaries are set within a margin of the previous fit, and any positive pixel within these windows is used to fit a second degree curve. Examples for the left and right lane lines are shown here:

![alt text][image11]
![alt text][image12]

Through experimental testing, curve estimation from scratch was found to perform more accurately than curve estimation from prior. Therefore, curve estimation from scratch is performed first on each image. If the fit curve is good enough, determined by the average residual value of the fit, this result is accepted. If the fit curve is not good enough, curve estimation from prior is performed next. If this average residual value is not good enough either, the fit from the previous image is used without modification. This previous fit will persist until an image is found with a high enough result for curve fit. This decision making happens in Lines 211-235, and the decision is made for each lane line separately.

Additionally, before drawing, the current fit is averaged with the previous fit to smooth the drawing of the lane in the output video (lines 218 and 231).

#### 5. Displaying lane information

Lane curvature and offset from center are displayed on each image. Curvature is calculated in lines 180-194 based on the lessons. Center offset is calculated (lines 248-252) as the difference between the lane center in pixels in the warped image and the mean lane line position taken from the curve fits calculated at the bottom of the image. This is transformed to meters using the ratio of pixels in the warped image lane width to an average lane size of 3.7m. The value displayed on the image is updated every 10 images to make it readable, and the value displayed is the average of the previous ten measures for both curvature and center offset.

#### 6. Drawing lane on an image

Using the fitted curves, a filled lane is drawn on the warped image and shifted back to the original perspective (lines 157-177). An example of the filled lane is shown here before and after the perspective shift:

![alt text][image5]
![alt text][image6]

---

### Video

Here's a [link to my video result](https://youtu.be/0oPe_fMTp50)

---

### Discussion

While this approach works reasonably well on the provided project video, it works pretty terribly on the challenge videos. One issue is that the binary thresholding gets really confused on the difference in tar color in the first challenge video. This could probably be helped by relying more on the lane line color and less on edge detection. I could also add a plausibility check for the width of the lane line which I'm not currently doing. 

Plausibility checks were a big issue for me in general. I tried comparing left lane line and right lane line curvature, but found that even in an image where both were detected well, the curvatures could vary a lot, making this a bad measure of a good fit. I also tried checking the curvature on a single side between successive images, and found this could also vary wildly. In the end, the best measure I found for checking plausibility was the average fit residual value, but I would guess there are better checks I could implement to determine good lane line fits. A check that the curves are at least facing the same direction could help.

In the challenge video, there is even a case where the fitted lane line intersect, which is clearly not right. A check should be added to make sure the lines don't intersect.

My pipeline would also probably have issues in really bright sunlight or varying shadow conditions. Some more work would need to be done on my thresholding.
