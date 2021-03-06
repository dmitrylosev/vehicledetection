
## Vehicle Detection Project

### Overview

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector. 
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[image1]: ./output_images/car_not_car.png
[image2]: ./output_images/hog_example.png
[image3_1]: ./output_images/sliding_windows_1_0.png
[image3_2]: ./output_images/sliding_windows_1_6.png
[image4_1]: ./output_images/find_cars_1.png
[image4_2]: ./output_images/find_cars_2.png
[image4_3]: ./output_images/find_cars_3.png
[image4_4]: ./output_images/find_cars_4.png
[image4_5]: ./output_images/find_cars_5.png
[image4_6]: ./output_images/find_cars_6.png
[image5_1]: ./output_images/bboxes_and_heat_1.png
[image5_2]: ./output_images/bboxes_and_heat_2.png
[image5_3]: ./output_images/bboxes_and_heat_3.png
[image5_4]: ./output_images/bboxes_and_heat_4.png
[image5_5]: ./output_images/bboxes_and_heat_5.png
[image5_6]: ./output_images/bboxes_and_heat_6.png
[image6]: ./output_images/labels_map.png
[image7]: ./output_images/output_bboxes.png
[video1]: ./project_video.mp4

### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.

The code for this step is contained in cell 1 lines 14 through 31 of the IPython notebook called [notebook.ipynb](./notebook.ipynb)).  

I started by reading in all the `vehicle` and `non-vehicle` images. I used glob() function to read images maching pattern at a time. Here is an example of one of each of the `vehicle` and `non-vehicle` classes:

![alt text][image1]

I then explored different `skimage.hog()` parameters. I grabbed random images from each of the two classes and displayed hog images to get a feel for what the `skimage.hog()` output looks like. I ended up with parameter ranges which have better visual results. Then I trained SVC classifier multiple times with predefind parameters. Winners were to have higher correlation of test accuracy and speed of the prediction:

| Orientations | Test Accuracy of SVC | 
| ------------ |:--------------------:|
| 3 | 0.984797 |
| 4 | 0.991273 |
| 5 | 0.988739 |
| 6 | 0.991836 | 
| 7 | 0.993525 | 
| 8 | 0.990991 |  
| 9 | 0.990709 |  
| 10 | 0.992962 |  
| 11 | **0.994651** |  
| 12 | 0.993806 | 
| 13 | 0.990709 |

| Pixels per cell | Test Accuracy of SVC | 
| ------------ |:--------------------:|
| 5 | 0.989865 |
| 6 | 0.991273 | 
| 7 | 0.991273 | 
| 8 | 0.990428 |  
| 9 | **0.99268** | 
| 10 | 0.992117 |  
| 11 | 0.986205 |
| 12 | 0.987894 |
| 13 | 0.98339 |

Higher number of pixes per cell performs higher prediction speed. Therefore `pixels_per_cell=(9, 9)` will give higher speed and accuracy. 

Also I compared one and all HOG channels, tried different color spaces and different cells per block. `All channels` performed better results. `HSV` and `YCrCb` performed better visual results than others. `cells_per_block=(2, 2)` works better than others.

| Parameter | Explored from | Explored to | Best value |
| --- | --- | --- | --- |
| orientations | 3 | 13 | **11** |
| pixels_per_cell | 5 | 13 | **9** |

Then I compared different spatial sizes, tested better results with different number of histogram bins and tested obtained better results with switched on Hog features. It was very often when spatial and histogram features generate many false positives and attempted to swithed off them completely. But it tunred out that SVC accuracy not reached 0.99% and I had to switch them on again.

The result of choosing spatial sizes and number of histogram bins are the following:

| Spatial size | Historgam bins | Hog features | Test Accuracy of SVC | 
| ------------ |:--------------------:|:--------------------:|:--------------------:|
| (8, 8)   | 64  | OFF | 0.957207 |
| (8, 8)   | 128 | OFF | 0.961712 |
| (8, 8)   | 128 | ON  | 0.992399, 0.990428 |
| (8, 8)   | 256 | OFF | 0.950169 |
| (16, 16) | 256 | OFF | 0.948761 |
| (32, 32) | 64  | OFF | 0.956081 |
| (32, 32) | 128 | OFF | 0.952703 |
| (32, 32) | 256 | OFF | 0.961993 |
| (32, 32) | 256 | ON  | **0.990709, 0.993243** |
| (64, 64) | 256 | OFF | 0.949043 |

And an example using the `YCrCb` color space and HOG parameters of `orientations=11`, `pixels_per_cell=(9, 9)` and `cells_per_block=(2, 2)`:

![alt text][image2]

#### 2. Explain how you settled on your final choice of HOG parameters.

I discovered that it also important to control the balance of features of different types. And ot turned out that 12 HOG orientations worked better after increasing the number of hist_bins and spatial_size features. The result of all experiments and intuition are the following:

```
color_space = 'YCrCb' # Can be RGB, HSV, LUV, HLS, YUV, YCrCb
orient = 12  # HOG orientations
pix_per_cell = 9 # HOG pixels per cell
cell_per_block = 2 # HOG cells per block
hog_channel = "ALL" # Can be 0, 1, 2, or "ALL"
spatial_size = (32, 32) # Spatial binning dimensions
hist_bins = 256    # Number of histogram bins
spatial_feat = True # Spatial features on or off
hist_feat = True # Histogram features on or off
hog_feat = True # HOG features on or off
```

#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

1. I concatenated HOG features, spatial and histogram features,
2. Normalized the data using StandarsScaler() method from sklearn package,
3. Shuffled and splitted the data into training and testing sets using train_test_split() method from the same package,
4. And trained a linear support vector machine classifier using LinearSVC() class

### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

I tested classifier with three window overlap options: 50%, 75%, 80%, visually compared results and selected 75% overlap as an option that showed better prediction results.

I then implemented HOG sub-sampling window search for extracting HOG features once per image using `cells_per_step=2` (overlap of 75%). The code for this step is contained in cell 1 lines 370 through 385 of the IPython notebook called [notebook.ipynb](./notebook.ipynb)).

![alt text][image3_1]
![alt text][image3_2]

#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

I separately searched on scales from 0.4 to 2.6 in order to get intuition of the quality of each scale prediction. Scales lower than 0.8 generated many false positives, which is hard to control by thresholding, scales from 0.8 to 1.0 generated moderate quantity of false positives and other scales generated contollable 0-1 false positives. I tried different combinations and stopped on series of scales `[0.6, 0.8, 1.4, 1.8, 2.6]`. 

You can see how pipeline is working on test images:

![alt text][image4_1]
![alt text][image4_2]
![alt text][image4_3]
![alt text][image4_4]
![alt text][image4_5]
![alt text][image4_6]

---

### Video Implementation

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)
Here's a [link to my video result (vehicles_project_video_final.mp4)](./vehicles_project_video_final.mp4)


#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.

The code for this step is contained in
* `process_frame(image, scales, scales_threshold)` method for filtering results of finding cars on every single frame
* `process_frames_bboxes(frames_bboxes, frames_threshold, image, visualize=True)` method for filtering reults when processing series of frames 

[notebook.ipynb](./notebook.ipynb)

I created a heatmap of an image, thresholded that map to identify vehicle positions, then used `scipy.ndimage.measurements.label()` to identify individual blobs in the heatmap and then assumed each blob corresponded to a vehicle. I constructed bounding boxes to cover the area of each blob detected.
    
Here's an example result showing the heatmap from a series of frames of video, the result of `scipy.ndimage.measurements.label()` and the bounding boxes then overlaid on the last frame of video:

### Here are six frames and their corresponding heatmaps:

![alt text][image5_1]
![alt text][image5_2]
![alt text][image5_3]
![alt text][image5_4]
![alt text][image5_5]
![alt text][image5_6]

### Here is the output of `scipy.ndimage.measurements.label()` on the integrated heatmap from all six frames:
![alt text][image6]

### Here the resulting bounding boxes are drawn onto the last frame in the series:
![alt text][image7]

---

### Discussion

#### What could be improved about the pipeline/algorithm

1. Add another kinds of features like `convex/concave`, `depth` etc.
2. Increase training dataset for higher prediction accuracy
3. Preprocess and augment training examples for robust vehicle detection in different road conditions (day/night, different road surfaces, shadows, lighting condition)
4. I would like to implement modern technics like YOLO and SSD which allow to increase detection quality and speed.
  
#### What hypothetical cases would cause pipeline to fail

1. Several false positives in one place
2. Tree shadows, road surface and other parts of images which are similar to vehicle
3. Small cars that is far from us
4. Mountanious terrain because we look at restricted area of an image

#### Some consideration of problems/issues faced

It was hard to find a compromise between possibility of detection of small size cars and quality of detection of middle and large size cars. The classifier was trained on 64x64 images and scales lower than 1.0 generated too many false positives that not always can be controled by thresholding.


```python

```
