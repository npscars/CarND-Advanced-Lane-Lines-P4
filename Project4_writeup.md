##Project Writeup 

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

First
* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images only once as we are using same camera for a video and use them for following steps.
Then create a pipeline to perform following tasks:
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/camera_cal.png "Output of Camera Calibration"
[image2]: ./output_images/interoutput_straight_lines1.png "Raw -> Binary -> Region Of Interest -> Warping"
[image3]: ./output_images/finaloutput_straight_lines1.png "Detect lanes lines algorithm output -> Final unwarped image"
[image4]: ./output_images/interoutput_test1.png "Raw -> Binary -> Region Of Interest -> Warping"
[image5]: ./output_images/interoutput_test4.png "Raw -> Binary -> Region Of Interest -> Warping"
[image6]: ./output_images/interoutput_test5.png "Raw -> Binary -> Region Of Interest -> Warping"
[image7]: ./output_images/finaloutput_test1_issue.png "Detect lanes lines algorithm output -> Final unwarped image"
[image8]: ./output_images/finaloutput_test1.png "Detect lanes lines algorithm output -> Final unwarped image"
[image9]: ./output_images/finaloutput_test4.png "Detect lanes lines algorithm output -> Final unwarped image"
[image10]: ./output_images/finaloutput_test5.png "Detect lanes lines algorithm output -> Final unwarped image"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second code cell of the IPython notebook located in "./CarND-Advanced-Lane-Finding-P4-Class.ipynb" 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpts` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpts` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpts` and `imgpts` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I also applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

The results look good as you can see that the image is undistorted especially if you focus on left end, right end and bottom of image where the car bumper is especially different between two images.

####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I tried several different options but finally chose only x gradient and color gradients for both white and yellow lines.

I found out that some things did not work:
1) y gradient was removing the far short lane lines especially when the image was of a turn in a lane.
2) I tried removing shadow from the image like with low value of l channel (0,25), but that was removing some lane sections especially in the video duration 22 to 22 and 38 to 39th seconds.
3) Did not see the benefit of using sobel magnitude and direction thresholds, actually that created quite few short lines in the middle of lanes. 

The functions for x gradient and color gradients are defined in 3rd code cell. It is used in function '_find_edges()', in 4th code cell. You can see the output of it in the first two subplots of image below.

![alt text][image2]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `corners_unwarp()`, which appears in the 5th code cell of the IPython notebook).  The function takes as inputs an image ('img'), camera matrix ('mtx'), distortion ceofficients (dist) and furthest view factor which I will describe later in the writup. 

I chose to automate the process of getting source points ('src') for calculating warp transform matrix. As the lanes are not always straight and car is not always in the middle of lanes it does make sense to go in that direction. 
Process of getting src points is coded in '_lane_lines_source_pts()' function as written in 4th code cell. It is an inspiration from the P1 project but with significant improvements to it. 
Essentially it does following steps:
1) using the binary edge image ('_edges_image'), mask the image to search the lanes lines only in area of interest.
2) use the cv2.HoughLinesP() to get probable lane lines
3) using slope methodology as in project P1, get the x points for all 4 polylines (left, right, bottom and top).
Some improvements done in this project:
1) Increased the threshold (i.e number of votes) for less noise and high possibility of a lane hough lines.
2) Check the hough line for not only minimum slope but also added maximum slope requirement as there were some times too many vertical lines but as the view is perspective it does not make sense.
3) Also added a default source points if the algorithm failed to find any specific src points for particular image. Default points were selected after considering the image size and most likely lane lines area.

After I get the source points (output as 'corners' from '_lane_lines_source_pts()' function), I select the 4 destination points in the warped image as :
dst = np.float32([[src[0,0], img_size[1]], [src[0,0], 0], [src[3,0], 0], [src[3,0], img_size[1]]]). img_size is essentially (720,1280) for this project and src[0,0] is minimum x value i.e. left lane line x value and src[3,0] is the maximum x value i.e. right lane line x value.

I then use the openCV functions cv2.warpPerspective() and cv2.getPerspectiveTransform(), to get the warped lane lines.
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear almost parallel in the warped image. As can be seen from in bottom two images of [image2] above.

There were certain challenges for getting the 'src' points correct for especially image which had turns, shadow and another car at right in next lane.

![alt text][image4]
![alt text][image5]
![alt text][image6]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

IPython code cell 7 is the algorithm for detecting Detect lane lines, determining the lane curvature and printing them in the video with detected lane lines defined in following 6 functions:
1) window_mask() -- only used to display to the lane masks in unwarped image.
2) overlay_window_centroids_with_image() -- only used as display function to check visually whether the algorithm of 'find_window_centroids() worked.
3) curverad_and_distance_tocentre() -- only used to display the output for  intermittent steps and not used in pipeline

4) find_window_centroids() -- this is similar to #Sliding Window Search# method which uses convolution to get number of hot pixels in windows as explained in the [class](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/a819d5fd-c00e-4e0e-b7d1-c1bee24b12a7):
But with two changes:
1) I added 100 pixels margin from center to get the first left lane center (l_center) and right lane center (r_center) in window search. Especially because in challenge video it was selecting middle concrete paved line as right lane line. This helped me get rid of that problem
2) I added 'if' condition as some of the windows were empty because of the way equation was defined it was taking - or + offset value. Instead it now takes last known value which to me makes more sense.

5) filtered_and_fit_laneline() -- input: this function takes in the binary warped edges image, window centroids found by find_window_centroids function.
function: Essentially filters the window centroids to consider only centroids found within 2 standard deviation and then uses those centroids to generate 2nd degree polynomial based lane lines.
output: filtered and fitted left and right lane lines x and y values.

6) image_with_laneline -- input: takes undistorted image, output of filtered_and_fit_laneline function, inverse perspective transform matrix and calculations of radius of curvature of left lane & right lane and distance of vehicle to center of lane.
function: This is taken from #Drawing# section of the following [class](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/7ee45090-7366-424b-885b-e5d38210958f)
One of the change is the addition of cv2.putText() function to put text on images or video about left curvature , right curvature and distance of vehicle to centre of lane. Calculation of these metrics are done in pipeline which is explain in next question.

The functions 4,5 and 6 also form the part of pipeline to run single/multiple images and as well videos.

Sample test images output from above functions:
![alt text][image8]
![alt text][image9]
![alt text][image10]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculated the radius of curvature in the pipeline in IPython code cell 10 and 11.
I also learned a great deal about class and how to use collections for smoothing the image. I would like to thank [Thomas Antony](https://github.com/thomasantony) for inspiring me to show how to use class for this project.

code cell 10 is where I define Lane class and its associated variables and methods.In this code cell and 11 the detection of lane lines and calculation of radius of curvature and position of vehicle with respect to center is done.

I hope explanations inside the pipeline code cells would be sufficient.

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

This is shown in previous section 4

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's my project video result:
"Result of project Video"
[video1](./output_videos/result_project_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

This project was also not easy. It took me some crazy weekends to finish it. Though now i am quite satisfied about the result. And also this was going to be used in next project and hence took more time to work on it.

I also performed the pipeline in both challenge videos and that led me to think about having different furthestview_factor as required by lane_lines_source_pts() function in code cell 4. 

What it is doing essentially is masking the images in y direction or, literally speaking, how much away from us. I felt like depending on speed of car this factor can be be reduced which means that the depth of view will increase if car speed is fast as it is assumed that the roads will be like in highway so quite long and when the speed is slow the depth of view will decrease like in city where there are sharp turns and hence lanes can be see not for huge distance.

So please see the results of challenge videos but I think this idea of changing factor can be automated if I know the speed of vehicle.

"Result of challenge video"
[video2](./output_videos/result_challenge_video.mp4)
"Result of harder challenge video"
[video3](./output_videos/result_harder_challenge_video.mp4)