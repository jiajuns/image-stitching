# image-stitching

There are different methods to do image stitching. Some of them are based on camera orientation and 3d reconstruction. But in this repo, I want to explain a method using image feature matching. This method contains 4 main steps:

1. Image features detections (DoG)
2. Feature Description (SIFT)
2. Keypoints matching
3. Compute Homography matrix between two frames
4. Wrap image and stitch image


## Setup environment

Create an virtualenv and use the requirements.txt to reconstruct environment:

    virtualenv -p python3 python3-env
    source python3-env/bin/activate
    pip install -r requirements.txt


## Image features detections (DoG)

In computer vision, there are many blob detection algorithm, including [Laplacian of Gaussian(LoG)](https://en.wikipedia.org/wiki/Blob_detection#The_Laplacian_of_Gaussian), [Difference of Gaussians(DoG)](https://en.wikipedia.org/wiki/Difference_of_Gaussians) and [Determinant of Hessian(DoH)](https://en.wikipedia.org/wiki/Blob_detection#The_determinant_of_the_Hessian). In order to understand other methods, it is important to have an intuitive understanding of Laplacian of Gaussian.

LoG can be thought as a two step process:
1. Gaussian filter to smooth out image
2. Apply Laplacian operator

[Here] (http://www.cse.psu.edu/~rtc12/CSE486/lecture11_6pp.pdf) is a good reference to know what LoG is doing. Given an image <a href="https://www.codecogs.com/eqnedit.php?latex=\inline&space;f(x,&space;y)" target="_blank"><img src="https://latex.codecogs.com/png.latex?\inline&space;f(x,&space;y)" title="f(x, y)" /></a>, and gaussian kernel <a href="https://www.codecogs.com/eqnedit.php?latex=\inline&space;g(x,&space;y,&space;t)&space;=&space;\frac{1}{2\pi&space;t}e^{-\frac{x^2ty^2}{2t}}" target="_blank"><img src="https://latex.codecogs.com/png.latex?\inline&space;g(x,&space;y,&space;t)&space;=&space;\frac{1}{2\pi&space;t}e^{-\frac{x^2ty^2}{2t}}" title="g(x, y, t) = \frac{1}{2\pi t}e^{-\frac{x^2ty^2}{2t}}" /></a>, the image can be convolved at a certain scale <a href="https://www.codecogs.com/eqnedit.php?latex=\inline&space;t" target="_blank"><img src="https://latex.codecogs.com/png.latex?\inline&space;t" title="t" /></a>. This will remove the high frequence noise in <a href="https://www.codecogs.com/eqnedit.php?latex=\inline&space;f(x,&space;y)" target="_blank"><img src="https://latex.codecogs.com/png.latex?\inline&space;f(x,&space;y)" title="f(x, y)" /></a>, LoG can be expressed as:

<a href="https://www.codecogs.com/eqnedit.php?latex=\bigtriangledown^2(g(x,&space;y,&space;t)&space;*&space;f(x,&space;y))&space;=&space;\bigtriangledown^2g(x,&space;y,&space;t)&space;*&space;f(x,&space;y)" target="_blank"><img src="https://latex.codecogs.com/png.latex?\bigtriangledown^2(g(x,&space;y,&space;t)&space;*&space;f(x,&space;y))&space;=&space;\bigtriangledown^2g(x,&space;y,&space;t)&space;*&space;f(x,&space;y)" title="\bigtriangledown^2(g(x, y, t) * f(x, y)) = \bigtriangledown^2g(x, y, t) * f(x, y)" /></a>

The reason behind that is the first derivative can be thought as edge detector. The second derivative can be thought as a blob detector. The LoG filter extrema locates blobs: maxima represents dark blobs on light background; and minima represents light blobs on dark background.

DoG is a approximation of LoG. LoG can be approximate by a difference of two Gaussian at different scales:
<a href="https://www.codecogs.com/eqnedit.php?latex=\bigtriangledown^2g_t&space;\approx&space;g_{t1}&space;-&space;g_{t2}" target="_blank"><img src="https://latex.codecogs.com/png.latex?\bigtriangledown^2g_t&space;\approx&space;g_{t1}&space;-&space;g_{t2}" title="\bigtriangledown^2g_t \approx g_{t1} - g_{t2}" /></a>

Below is an visualization of this algorithm:

![alt text](https://docs.opencv.org/3.1.0/sift_dog.jpg)

Once DoG is computed, the algorithm will search for local extrema over difference scale in space. For eg, one pixel in an image is compared with its 8 neighbours as well as 9 pixels in next scale and 9 pixels in previous scales. If it is a local extrema, it is a potential keypoint. It basically means that keypoint is best represented in that scale. It is shown in below image:

![alt text](https://docs.opencv.org/3.1.0/sift_local_extrema.jpg)

## Feature Description (SIFT)