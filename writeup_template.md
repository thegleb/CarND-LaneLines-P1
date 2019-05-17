# **Finding Lane Lines on the Road** 

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[masked_grayscale]: ./writeup_images/masked_grayscale.png "Masked Grayscale"
[canny_left]: ./writeup_images/canny_left.png "Left lane line after canny edge detection"
[canny_right]: ./writeup_images/canny_right.png "Right lane line after canny edge detection"
[final_lane_lines]: ./writeup_images/final_lane_lines.png "Lane lines overlaying an image"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

I broke down the problem into several steps.

#### 1) Image pre-processing

First, we want to filter out all the non-relevant image information. Since we are looking for white and yellow elements of lane lines (e.g. solid line or dotted segments), we create masks for yellow and white colors. One approach that makes this a bit simpler is converting the original RGB image to the HSV color space (hue, saturation, value). This allows us to target pixels with colors located in some specific part of the color spectrum as well as pixels that have specific saturation/value values.

Normally hue values will range from 0 to 359 but with OpenCV we only get a range from 0 to 179 which was something I discovered through trial and error and some frustrated googling. I also extended the range of yellows I was looking for to include some colors nearby to make up for images/video that may not have the right color balance. With white colors I increased the accepted saturation values a bit to make sure we capture the white lane markers on the lighter part of asphalt in the `challenge.mp4` video.

We also create a copy of the original image in grayscale to make edge detection easier.

Then we use the color masks on the grayscale image to achieve something like the image below:

![Masked grayscale image][masked_grayscale]

#### 2) Create shape masks, one per lane line

I decided to process left lane lines separate from the right lane lines just to avoid having to do some additional filtering in case my original pipeline was too noisy. My solution didn't end up "needing" it due to the pre-processing step filtering out most of the noise I was seeing early on, but I kept the separation anyway. These masks are in the shape of somewhat skewed trapezoids, and we use the `region_of_interest` helper to return a version of the grayscale image where only the masked parts are visible and the rest is black.

#### 3) Canny edge detection

After getting the masked left/right lane line images, we run canny edge detection to get edges for the lane line or lane line fragments:

![Canny edges for the left lane line][canny_left]
![Canny edges for the right lane line][canny_right]

#### 4) Hough line transform

With the canny edge images, hough line transform becomes fairly meaningful. We apply this step next for the left and right lines. The result is an array of line segments.

#### 5) Filter out / average together the resulting line segments

For this part I applied a somewhat brute force method that I'm not super proud of because it feels clumsy and is definitely not optimized. Here is the step by step:
* Iterate over the line segments returned in 4)
* Separate the line segments into points
* Merge clusters of points within ~50 pixels of each other (exact distance determined by trial and error)
* Now we have pretty much a contiguous segment that may not be quite straight
* Take the start and end points and draw a line throught them, constraining it to minimum/maximum Y values to make sure we place the lane lines in the right place in the image

#### 6) Do some caching and averaging of the resulting lines

This step is sort of mixed into the logic of step 5 just to avoid repetition, but the idea is this:
* If we haven't detected any lane lines, then simply push the hough line segments from 4) into an array and process it
* If we've already done the step above once (e.g. this is frame #2) then perform step 5 once for the previous hough lines and once for the current hough lines
* Push the start/end points into separate arrays
* Average the start/end points of the last N frames
* Return the resulting averaged start/end point

This methodology also has a side effect of displaying a previously detected lane line in case the detection in the current frame doesn't work as expected.

#### 7) Now draw the line segments and overlay them over the original image

![Final lane lines detection][final_lane_lines]


### 2. Identify potential shortcomings with your current pipeline

* Color masking isn't perfect so we do lose lane lines briefly in the `challenge.mp4` video particularly when dealing with shadows near the end
* Lane curvature is ignored because we assume lane lines are perfectly straight, which works for videos of the car moving in a straight line but will fail for curve segments (you can see this a bit in the challenge video, because the lines don't necessarily follow the road in some cases)
* The averaging step isn't super optimized and we will re-process the previously detected hough transform lines for every frame. For example, we average lane lines from 3 frames, meaning the 2 previous frames will be re-computed each time together with the 3rd frame.

### 3. Suggest possible improvements to your pipeline

* I think the biggest improvement would be to compute Bezier curves for lane lines instead of assuming they are straight.
* Another improvement might be to process the image a little more before doing edge detections on it - this would help in segments of the challenge video where the lane markings aren't as sharply differentiated from the road around them