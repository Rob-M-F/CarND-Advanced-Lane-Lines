## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1a]: ./writeup/distorted_overhead.png "Distorted Overhead"
[image1b]: ./writeup/undistorted_overhead.png "Undistorted Overhead"
[image2a]: ./writeup/distorted_test3.jpg "Distorted Test Image 3"
[image2b]: ./writeup/undistorted_test3.jpg "Undistorted Test Image 3"
[image3a]: ./writeup/sobel_x.jpg "Horizontal Absolute Sobel"
[image3b]: ./writeup/sobel_y.jpg "Horizontal Absolute Sobel"
[image3c]: ./writeup/hsl_l.jpg "HSL Channel L"
[image3d]: ./writeup/mask.jpg "Combined Mask"
[image4a]: ./writeup/unwarped_straight_lines1.jpg "Unwarped Straight Lines"
[image4b]: ./writeup/warped_straight_lines1.jpg "Warped Straight Lines"
[image5a]: ./writeup/search_image.jpg "Sliding Window Search"
[image5b]: ./writeup/drawn_lane.jpg "Drawn Lane"
[image6]: ./writeup/result.jpg "Final result"
[image7]: ./writeup/drawn_lane.jpg "Drawn Lane"
[video1]: ./writeup/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

## Writeup / README

The project notebook referenced throughout this writeup can be found at "./Advanced-Lane-Lines.ipynb".  

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.  

To associate the camera calibration data with the associated images in need of undistortion, I created the Camera class found in the second cell of the project notebook.  

Upon instantiation, the camera self-calibrates using the calibration function. This function uses OpenCV to perform the required steps for camera calibration. Each calibration image is converted to grayscale using OpenCV's cvtColor. The findChessboardCorners function operates on the grayscale image to produce the chessboard corners list which is added to img_list for later use, at the same time, the equivalent 3d points are stored in obj_list. The obj_list points assume the chessboard is attached to a fixed plane, preventing changes in the z coordinate.  
Once all of the calibration images have been processed, the function calls a final OpenCV function, calibrateCamera, to generate the calibration and distortion coefficients, which are stored in the Camera instance as mtx and dist.  

See the test image below for an example distorted image and it's undistorted equivalent:  

![Distorted][image1a]
![Undistorted][image1b]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The pipeline accesses images using a generator function, Camera.image, in the Camera class. This function undistorts the images before passing them on. Below is the before and after of image test3 used for tuning the pipeline.  
![Distorted][image2a]
![Undistorted][image2b]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used absolute sobel thresholding and channel thresholding on the s channel of the hls colorspace to isolate the lane lines. The 5th cell of the project notebook contains all of the function definitions for my thresholding process.  
Within the absolute_sobel function I use a BGR(Blue, Green, Red) to Grayscale color transform followed by histogram equalization to prepare the image for thresholding. The function applies a horizontal (x-axis) amdvertical (y-axis) Sobel gradient operation on the image in separate function executions. The absolute value of the resulting gradient map is then scaled up to a maximum value of 255, which is then passed through the final threshold operation, which creates a map showing ones everywhere the scaled up values are within the threshold values. All other pixels are set to zero.  
The hls_select function converts the BGR image to the HSL colorspace, then drops all but the Lightness channel, denoted as 'l'. This channel is thresholded as before, with only pixels with values within the threshold range set to one and all others set to zero.  

My final thresholding mask uses all three described masks:
    Horizontal sobel kernel 13 pixels wide and thresholds at 20 and 120.
    Vertical sobel kernel 17 pixels wide and thresholds at 16 and 128.
    HSL Channel L thresholded at 110 and 255.
    The Horizontal and Vertical Sobel kernels are combined in a logical AND, creating a new mask with ones only where both previous masks had ones, and zeros elsewhere.
    The combined Sobel mask was then combined in a logical OR with Channel L, creating the final mask with ones everywhere either mask had ones, with zeros everywhere else.

![Horizontal Sobel][image3a]![Vertical Sobel][image3b]![HSL Channel L][image3c]![Final Mask][image3d]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

To handle the perspective transformations, I created a Perspective Transform class. This class accepts Source and Destination point arrays and creates the transformation and inverse transformation mapping. These are stored in the object to simplify use. My hand selected point arrays are provided as the default value in the constructor. The default mapping is shown below:
| Source        | Destination   | 
|:-------------:|:-------------:| 
| 570, 450      | 50, 50        | 
| 730, 450      | 1230, 50      |
| 260, 600      | 50, 700       |
| 1055, 600     | 1230, 70      |
The class has a single apply member function that, by default, applys the transform. When the function is also passed an argument of reverse=True, it applys the inverse transformation, resulting in the original image perspective. This class occupies the 3rd cell in the project notebook. The transformation of straight lane lines is shown below.  
![Unwarped][image4a]![Warped][image4b]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Fitting the lane, and therefore the lines, occurred within the Lane class, defined on the 4th cell of the project notebook. The class uses lists to track 103 points along the curve for up to the last 10 frames. The primary function is search_lines which accepts a single channel image. This image is then passed to the find_base function to apply a histogram search for the lane lines. This function searches the left 1/3 of the image for the left line and the right 1/3 of the image for the right line. The peak of each segment is returned to the search_lines function as the starting points for the line searches.  

The search then finds the average marked pixel within a window 1/100th of the image tall and chk_rng wide to either side of the previous line center. This search is repeated as it works up the image. Whenever the search returns a value of zero, likely indicating blank space, the line mirrors the findings of its partner. Once the search is complete, the identified points are added as a numpy array to the list for that line.  

Separately, when the pipeline is ready to draw the lane, compile_line averages across each y-coordinate, for each line.  This average lane line position is then fit to a curve by fit_line. Finally, get_points uses the fit curve to generate new a new point for every y-coordinate in the final image.  

![Line Search Output][image5a]![Drawn Lane][image5a]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Within the Lane class, the calculations for radius of curvature and vehicle position are handled within the find_stats member function. The radius of curvature is calculated using the get_points function with the associated fit_line function. The calculations are completed with the pixel values adjusted to measurements in feet.  

Position in the lane assumes the center of the image is the center of the car. This point is calculated in feet. Separately, the width of the lane at the front of the car is calculated, again in feet. Adding half the lane width the the position of the left lane marker gives an approximation of the lane center. Subtracting this lane center from the car center gives the car offset from lane center.  

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

With the Lane.draw function rendering out the lane, it is left to the pipeline itself, in cell 6 of the project notebook, to combine the original image and the unwarped lane image using the OpenCV addWeighted function. The lane overlay is added with 30% weight.  

![Final overlayed image][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./writeup/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I found that my lack of in-depth experience with OpenCV hindered my efforts. In addition, I found threshold filter adjustment to be a complex and slow process.  
My pipeline would handle lane changes very poorly. It would also fail in inclement weather and night time driving. I believe additional filters tuned over a wider range of driving conditions would improve performance in these conditions. Unfortunately, lane changes would require rebuilding the model to expect more than 2 lane lines.  
