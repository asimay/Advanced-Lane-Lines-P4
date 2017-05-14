
## **Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

![alt final][image12]

[//]: # (Image References)

[image1]: ./output_images/findanddrawChessboardCorners.png "findcorner"
[image2]: ./output_images/undistort.png "undistort"
[image3]: ./output_images/transimg.png "transimg"
[image4]: ./output_images/test4undist.png "undist Example"
[image5]: ./output_images/combine.png "combine the color"
[image6]: ./output_images/pers_warp.png "perspective and wrap"
[image7]: ./output_images/curveline.png "curve line perspective and wrap"
[image8]: ./output_images/2fit.png "fit the line"
[image9]: ./output_images/histgram.png "histgram.png"
[image10]: ./output_images/formular.png "formular.png"
[image11]: ./output_images/final.png "final.png"
[image12]: ./output_images/final.gif "final.gif"
[image13]: ./output_images/finalr.jpg "final.gif"

[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state of camera calibration:

The code for this step is contained in the 4th code cell of the IPython notebook located in "./CarND-Advanced-Lane-Lines.ipynb" .

first, read all the calibration images of the camera, and then  prepare "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  

![alt findcorner][image1]

I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt undistort][image2]

And then I choose 1 image to test the perspective transform is work or not. this will use `cv2.getPerspectiveTransform` to get perspective transform Matrix M, and then use `cv2.warpPerspective` to warp the image. result shows below, it works, so we can proceed next step.

![alt transimg][image3]

### Pipeline (single images)

#### 1. Original image vs. distortion-corrected image:

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

![alt text][image4]

#### 2. Color transforms and gradients:

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at section of 'Use color transforms, gradients, etc., to create a thresholded binary image'.).  

I used sobel_x threshshold in function `abs_sobel_thresh()`, and used HLS color space to get feature in function `hls_select()`, and then combined both of them.

Here's an example of my output for this step, we can see we get good line of road:

![alt text][image5]

#### 3. Perspective transform:

The code for my perspective transform includes a function called `images_warp()`, which appears in section of 'explore perspective transform and wraped image' in jupyter notebook.  
The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [((img_size[0] / 6) - 10), img_size[1]],
	[[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
	[(img_size[0] / 2 + 55), img_size[1] / 2 + 100]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]])

dst = np.float32(
    [(img_size[0] / 4), img_size[1]],
	[[(img_size[0] / 4), 0],
	[(img_size[0] * 3 / 4), 0]],
    [(img_size[0] * 3 / 4), img_size[1]] )
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 204, 720      | 320, 720      |
| 585, 460      | 320, 0        | 
| 705, 460      | 960, 0        |
| 1110, 720     | 960, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image6]

And then I used a curved line to test the perspective transform, and it works well.

![alt text][image7]

#### 4. Identified lane-line pixels and fit their positions with a polynomial:

Then I did some other stuff and fit my lane lines with a 2nd order polynomial.
1. use `getHistogram(warp_img)` function to get histogram of the warped binary image, this can get the lines positon in image. I translated the histogram by `dst_pts[0][0]-200 = 120` pix from left to right, only process the histogram between the left line and right line and with some pixes buffer, because when road has much shade this will impact the handling result, so the histogram range from (120 ~ 1160) now, result like this:

![alt text][image9]

2. use slide window method in function `fit_poly()` to calculate the slide window's width, height, coordinate of window's border, and for each loop of slide windows, get the good lines index, and we will use these point to further procedure.

3. use the point above to fit the lines with a 2nd order polynomial:

```python
left_fit = np.polyfit(lefty, leftx, 2)
right_fit = np.polyfit(righty, rightx, 2)
    
ploty = np.linspace(0, warp_img_h.shape[0], warp_img_h.shape[0] )
left_fitx = left_fit[0]*ploty**2 + left_fit[1]*ploty + left_fit[2]
right_fitx = right_fit[0]*ploty**2 + right_fit[1]*ploty + right_fit[2]
```

and then visualization the result, like this:

![alt text][image8]

#### 5. Calculated the radius of curvature of the lane and the position of the vehicle with respect to center:

1. transform the good line points from image pix space to meter space, I simply use this transform formular:

![alt text][image10]

and code:

```python
ym_per_pix = 30/720  # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension 
```

and the re-fit the line from 'meter' perspective. This will help us to get radius of curvature of lane.

About the position of the vehicle with respect to center, I assume the camera is mounted at the center of the car and the deviation of the midpoint of the lane from the center of the image, this offset is what i'm looking for.
I calculate the offset between the image center and lane center, and switch to meter metric.
lane center is calculate by substract leftx_fix and rightx_fix after fit the 2nd order polynomial.

```
distance_from_center = np.absolute((img_size[0]/ 2 - lane_center) * xm_per_pix)
```

#### 6. Result plotted back down onto the road such that the lane area is identified clearly.

1. we need draw the fit lines on image, and then use inverse perspective to plot it back down onto road.

Here is an example of my result on a test image:

![alt text][image11]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here is an example of my result on a video:

![alt text][image13]

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

1. source point and destination point are choosed not correctly, it will greatly affect the effect of perspective transform. It will cause the line looks not parellel of transform result. So we need a tuned point or some other algorithm to calculate the point well.

2. HLS color space, because the road has shade and lightness, the color space choosen is also very important.

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
