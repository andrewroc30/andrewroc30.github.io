---
layout: post
title: "Mandelbrot Fractal Zoomer"
date: 2019-02-10
---

<video width="100%" height="100%" controls>
  <source src="/assets/videos/mandelbrot_430img_1_01zoom_1000iterations.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

# Description

This is a program which creates videos of zooming into a single point on the [Mandelbrot fractal set](https://en.wikipedia.org/wiki/Mandelbrot_set).  After downloading the project, and installing stated dependencies, you can choose the amount of images you want to create, the zooming rate, and the center point.  Here is an image of the UI (old) with all of the options:

<img src="/assets/images/mandelbrot-fractal-zoomer-ui.png" width="50%" height="70%" style="display: block; margin-left: auto; margin-right: auto;">

Along with creating images and animations, this program has a mode for exploring the fractal, allowing you to zoom in on a dragged box (similar to [this site](http://math.hws.edu/eck/js/mandelbrot/MB.html)).

# Algorithm

The algorithm I used is the 'escape-time' algorithm, where each point is calculated a given number of times and a color is assigned based on the output.  By using this algorithm, a single image can be created.  In order to create a video of zooming into the image, I generate several images each with various zoom, which is done by dividing the x and y coordinates by the given zoom for each point, so as the zoom increases, each point calculated will be closer to the center point, effectively zooming into the image.

After creating the image as a BufferedImage, it can be saved as a PNG.  After saving the images they can then be converted to a GIF using Java code, or converted to an MP4 using FFMPEG.

## Smooth coloring

I used a smooth coloring algorithm on top of this as well.  Normally, in each of these images, you get quite a bit of banding, tainting the aesthetics.  This is due to the way that the escape-time algorithm works.  Since each point has a single iteration where it escapes, and each iteration is assigned a single color, we get 'bands' of colors, where it all of a sudden switches from one iteration to the next.  By using linear interpolation on the rgb values, we can smooth out the colors so that there isn't such a sharp difference between the bands.  This was a bit harder than I had originally anticipated, and I ended up learning how rgb values are stored in a single integer, as well as how to read and store data directly through bytes instead of just using set data types.  In the process of figuring this out, I sometimes got weird but interesting results like this:

<img src="/assets/images/mandelbrot-incorrect.png" width="50%" style="display: block; margin-left: auto; margin-right: auto;">

The final results can be seen here, going from a normal banded image to the new smooth image.

<img src="/assets/images/mandelbrot-banded.png" width="50%" style="display: block; margin-left: auto; margin-right: auto;">
<img src="/assets/images/mandelbrot-smooth.png" width="50%" style="display: block; margin-left: auto; margin-right: auto;">

## Performance
By trying to create images that are more and more zoomed in, the runtime of the algorithm is increased greatly.  This is mainly due to 2 things:

  1) The time taken for arbitrary precision calculations.  While this is not implemented yet, having played around with the BigDecimal class in this makes it clear that the time taken with these calculations is immense, and only grows the more precise the calculations need to be.  Right now, while using doubles in Java, the program can only go so deep before running into an issue with the lack of precision with doubles.  By this, I mean that the data type can only hold so many significant digits, and there comes a point when all the points on a zoomed screen will be approximated to the same value, as shown by this video:
  
<video width="100%" height="100%" controls>
  <source src="/assets/videos/mandelbrot-double-limit.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
  
  2) The iterations required to determine whether or not a point is or isn't in the set.  As we zoom into the image, we are effectively creating a higher resolution image, but are calculating a smaller portion of it, so the precision required for each point increases as we zoom further, which means that we need to have more iterations per point.  For reference, this is what it looks like when there are 10 vs 100 iterations for the initial image.

<img src="/assets/images/mandelbrot-10-iterations.png" width="50%" style="display: block; margin-left: auto; margin-right: auto;">
<img src="/assets/images/mandelbrot-100-iterations.png" width="50%" style="display: block; margin-left: auto; margin-right: auto;">

## Performance Optimizations
The runtime was slowed down considerably by the zooming, so there were a couple of things that I tried and a couple of things that I will try.

First, I decided to make the program multi-threaded.  By doing this, the program could be creating multiple images simultaneously (dependent on the CPU), which sped up the program dramatically.  There were several issues that cropped up when doing this, as making parallel programs does often.  For one thing, there were some race conditions that I had when doing this, since I initially put the images into an array, and each thread wrote the completed image to that array.  This was fixed by the addition of a lock, which worked like a charm.  Then, when trying to create a larger video, I ran out of memory, since all of the threads were created at once and each one was trying to hold its own image in memory.  This was fixed by keeping track of the number of active threads, and keeping them down to a manageable amount so that there would never be too much data in memory at once.  While this sped up my program considerably, full utilization of resources had me concerned about the intense CPU usage, since I don't want it running at full capacity for extended periods of time.  I don't really enjoy seeing this image...

<img src="/assets/images/full-cpu.png" width="50%" style="display: block; margin-left: auto; margin-right: auto;">

Second, I used cardioid checking.  Cardioid checking simply checks whether or not the current point is in the main cardioid.  By doing this, we can forgo running the iterations on some of the points, which speeds up the program considerably as well, in certain situations.  If the image doesn't contain any points which satisfy this, then this speedup does nothing, but a point on the edge of the cardioid will be helped greatly, especially for the initial images.

Third, I tried/am trying to use the GPU as a speedup.  The calculations for obtaining a single point in an image is strictly based upon lots of floating point calculations (barring the coloring, which I can set based on the returned GPU calculation).  This led me to the idea of using a GPU for these calculations, since they excel at FLOPS, which is exactly what I need.  By doing this, I can potentially calculate each point individually in parallel in the GPU, and the calculations themselves should hopefully be faster as well.  Not only this, but stress will be taken off of the CPU.  By creating several images at once with multithreading, I've gotten to the point where the CPU is running at 100% on all cores, which I would definitely prefer not to do for long periods of time, and pushing work from the CPU onto the GPU would help with this.  Unfortunately, I haven't gotten this to work yet, mainly with issues surrounding getting code integrated with CUDA, but have high hopes for performance if I ever complete this.

Fourth, I will try increasing the iterations as needed when zooming into the image.  Since we will need higher iterations as we zoom further in, the intial images created will not need as many iterations as the later images.  So, by increasing the iterations as we zoom, the initial images can be created much faster.  Unfortunately, there doesn't seem to be any easy way to do this, like a formula.  However, it should be possible to estimate based upon the number of same-colored points (points that escape in the same iteration).  This isn't a huge priority, since a lot of the high precision calculations at the higher zooms are what the greater portion of the runtime are dedicated to, so reducing the iterations for the initial images certainly helps with the overall runtime, but will be less and less effective as we get further into the set.

Fifth, I will generate only key frames and then fill in intermediary images.  If I only generate the key frames, like for example one image every 2x zoom, and then fill in the frames that are in between without recalculating each image, I can save a lot of runtime by not having to calculate an image for every single frame.  While this would be a considerable speedup, I have concerns about the quality of the video using this technique, since there would be less detail in between key frames, to the point where it may be noticable, especially if the zoom difference between the frames is too large.

## Code

All of my code is freely available at this link: [github.com/andrewroc30/Mandelbrot-Fractal-Zoomer](https://github.com/andrewroc30/Mandelbrot-Fractal-Zoomer)
