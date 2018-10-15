# image-stitching

There are different methods to do image stitching. Some of them are based on camera orientation and 3d reconstruction. But in this repo, I want to explain a method using image feature matching. This method contains 4 main steps:

1. Image feature(keypoints) detections
2. Keypoints matching
3. Compute Homography matrix between two frames
4. Wrap image and stitch image


# Setup environment