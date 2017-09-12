# **Finding Lane Lines on the Road** 

# Writeup & Project description
### Joris van Tatenhove

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[Mask]: ./examples/mask.jpg "Masking the image"

---

# 1. Conversion pipeline

In this chapter, we will explained the techniques used to detect lane lines on a road. One by one, we will go through the pipeline of going from a static image of a road to an image where we just have the lines that we need to be able to drive in the middle of our lane.

## Preprocessing the image

First off, we convert our image to grayscale. In the process of finding lane lines, we are only interested in lines that separate 'light' and 'dark' areas of the picture. The actual colors are not relevant to us at all.
Then, to be able to deal with some noisy data, such as small particles on the road, we apply a gaussian blur to the picture. This will help smoothen the picture, so we only find the areas that are important to us.

## Edge detection

To detect the edges, we use the Canny edge detection algorithm. This algorithm first applies some additional blurring steps, and then identifies all so called 'edge pixels'. An edge pixel is a pixel where the color gradient, so the difference between the color of this pixel and its surrounding pixels, is 'high enough'. What constitutes 'high enough' is one of the parameters of the algorithm that we can tweak, and we will play around with this value to suit our needs.

## Masking the image

Unfortunately, just identifying all edges is not good enough. We need to be able to separate the edge of the lane line from the edge of a nearby car, bridge or traffic light. Luckily for us, we have some rules of thumb that can help us in detecting the edges of the lane line.

One of these is that we know where we can expect the lane lines. We usually drive within two lane lines, so we can expect them to start from the bottom left and the bottom right of our image frame, and then point into the direction of the horizon, right in front of us.

This means that we can reasonably expect our lines to be somewhere in the following area:

![The region of the image we look for lane lines][Mask]

## Hough transformation

Something else that we can use, is that we want to detect lines that are relatively long. Some lane lines are just completely connected, but also the dashed lines still are made of line parts with decent length. We can use a Hough transformation to filter out all the edges that are not long enough to be a lane line.

Hough transformation translates an edge to a point. An edge is defined by the formula `y = mx + b`. This means that a line is uniquely defined by the values of `m` and `b`. That means that we can represent a line in `(x, y)`-space by a point in the `(m, b)`-space. This is called the _Hough space_. Similarly, a point in `(x, y)` coordinates coincides with a line in Hough space. 

These lines in Hough space have one very nice property. If we have some points that are on a straight line in `(x, y)`-space, they will have intersecting lines in Hough space! Counting the number of lines going through a point in Hough space gives us information about the length of the line in the original image. 

Of course, because we work in pixels, we will lose some precision. We can work around this by drawing a grid over our Hough space, and pretend two lines intersect if they are in the same rectangle in our grid. 

As a post processing step, the lines that we are left with can still be filtered out. We define a `minimum line length`, and also a `maximum line gap`: if two lines are really close to each other, it is probable that it is actually a single line in real life, but a small object separated the two line parts.

Tweaking these parameters, and also the size of the grid and the number of intersections required, will help us detecting the line edges.

## Drawing the lane lines

Now that we have hopelly found edges we are interested in, we still need to translate them into lane lines. We need to somehow group together the dashed lane lines. In order to do this, we again look into the mathematics of a line.

The Hough transformation procudure outputs a set of line segments, where line a line segment is defined by a starting point `(x1, y1)` and `(x2, y2)`. We can derive the slope of this line: `m = (y2 - y1)/(x2 - x1)`. We can derive some information about a line based on the slope.

First of all, we use another rule of thumb: lines with an absolute value of the slope `abs(m)` of less than `0.5` will not be lane lines: we know that the lane lines are almost diagonal lines heading towards the horizon. This will filter out lines parallel to the horizon.
Also, lines with a positive slope are way more likely to belong to the right lane line, and lines with a negative slope more often belong to the left lane line.

Now that we've grouped all identified edges into discarded edges, left lane edges and right lane edges, we can finally start constructing the lane lines! 

We average the midpoint of all lines belonging to the right lane line, and also average the slope of these lines. 

The average midpoint and average slope again gives us a line. From this line, we take the segment that starts at the bottom of the picture, and extend this to the horizon. We can use the same procedure to find the left lane line.


# 2. Results, discussion and potential improvements

I've uploaded the result to YouTube (also available in the folder `/test_videos_output`). I've provided a total of five videos:

## solidWhiteRight

[This](https://youtu.be/UKiDHQFWzeY) video shows the identified edges for the input video `solidWhiteRight`, and [this](https://youtu.be/IBScCDvvbFE) video shows the lane lines derived from the edges.

We can see how the dashed line on the left is identified as a single line when they are far enough from our point of view. This is due to the line gap that we use in the Hough algorithm. The right side of the right edge is apparently somewhat difficult to discover, but that is not needed to derive the lane lines, as can derive enough information from the line edges that are far away. We do see that mainly the left lane line is kind of wobbly: a slight difference in slope can have a pretty big impact on the endpoints of the derived line. I'm not quite sure _how_ we exactly use the lane lines in practice to center our vehicle, but this could potentially be a problem that we'd need to tackle.

## solidYellowLeft

[Here](https://youtu.be/4jS0_yhQ1OQ), we can find the detected edges for `solidYellowLeft`. The lane lines we've calculated are shown [here](https://youtu.be/9_L34K5Y3q0).

We again see that we pretty much glue together the dashed lines that are far away, and only occasionally pick up the line segments that are closer to us. Again, the dashed edges give us a wiggling line, this time on the right side. Because we mostly find edges close to the horizon, the midpoint is usually there and a small difference in the average slope yields quite some pixels offset on the bottom of our frame. 
Detecting a yellow line does not appear to be harder than detecting a white line.


## challenge

We also tried detectinng the edges on the more challenging input video. This is very interesting, as it points out quite some areas the algorithm we've created should improve upon. The result can be found [here](https://youtu.be/3ioaXGisrQA). 

This video clearly points out some areas we are underperforming:

- Detecting nearby dashed lines

Like in the previous videos, we do not often detect line segments of a dashed line close to the vehicle. Unlike previously, this is a problem, because we also do not detect the lines far away, as we are taking a relatively sharp turn. If I had more time, I would tweak the parameters of the Hough transformation, as we probably filter the edges out. 

- Detecting the yellow lane line on a light road

In the middle section of the video, we do not detect the yellow lane lines when the underlying road becomes almost beige in color. This can be tackled by improving the parameters for the Canny edge detection algorithm: the amount of blur and the thresholds should be fine tuned so we do detect these edges.

- Detecting lines outside of the road

We detect many lines that are not part of the road. The lines are mostly found in the trees and their shadows. We can deal with the trees by improving our mask. The shadows of the trees could be filtered out by finetuning the parameters of the Hough algorithm: the minimum line length, maximum gap and the threshold play a huge role in detecting these line edges.