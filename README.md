# image-stitching

There are different methods to do image stitching. Some of them are based on camera orientation and 3d reconstruction. But in this repo, I want to explain a method using image feature matching. This method contains 4 main steps:

1. Image features detections (DoG)
2. Feature Description (SIFT)
3. Keypoints matching
4. Compute Homography matrix between two frames
5. Wrap image and stitch image


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
Key locations are defined as maxima and minima of the result of difference of Gaussians function applied in scale space to a series of smoothed and resampled images. Low-contrast candidate points and edge response points along an edge are discarded. Dominant orientations are assigned to localized keypoints.

![alt text](https://qph.fs.quoracdn.net/main-qimg-a471637e843649a6b9483bcc70ba62c1.webp)

The figure above illustrates the computation of the keypoint descriptor. First the image gradient mangnitudes and orientations are sampled around the keypoint location, as shown on the left. They are then weighted by a Gaussian window, indicate by the overlaid circle. In order to achieve orientation invariance, the coordinates of the descriptor and gradient orientations are rotated relative to the keypoint orientation. These samples are then accumulated into orientation histograms summarizing the contents over 4*4 subregions.

The descriptor is formed from a vector containing the values of all the orientation histogram entries, corresponding to the lenghts of the arrows on the right side of above figure. For example, vector of this figure has '2x2x8=32' elements. In order to reduce the effects of illumination change, the vector is normalized to unit length. Another detail is reduce the influence of large gradient magnitudes by thresholding the values in the unit feature vector to no larger than 0.2, and then renormalizing to unit length.

## Keypoint Matching
The idea behind keypoint matching is to take the descriptor of one feature in first set and is matched with all other features in second set using some distance calculation. And the closest one is returned. In our code we use [flann](https://docs.opencv.org/2.4/modules/flann/doc/flann_fast_approximate_nearest_neighbor_search.html#id1) provided by opencv.

## Compute Homography matrix
Homography matrix is a special case of fundamental matrix, which applies to scenes which are planar or far away. A planar homography H satisfies the constraint x' = Hx (x' represents the second image; and x represent the first image). H is a 3x3 matrix of rank 3 in which there is a one-to-one point correspondence between the first and second images. 