## Project: Perception Pick & Place
---
# Required Steps for a Passing Submission:
1. Extract features and train an SVM model on new objects (see `pick_list_*.yaml` in `/pr2_robot/config/` for the list of models you'll be trying to identify). 
2. Write a ROS node and subscribe to `/pr2/world/points` topic. This topic contains noisy point cloud data that you must work with.
3. Use filtering and RANSAC plane fitting to isolate the objects of interest from the rest of the scene.
4. Apply Euclidean clustering to create separate clusters for individual items.
5. Perform object recognition on these objects and assign them labels (markers in RViz).
6. Calculate the centroid (average in x, y and z) of the set of points belonging to that each object.
7. Create ROS messages containing the details of each object (name, pick_pose, etc.) and write these messages out to `.yaml` files, one for each of the 3 scenarios (`test1-3.world` in `/pr2_robot/worlds/`).  [See the example `output.yaml` for details on what the output should look like.](https://github.com/udacity/RoboND-Perception-Project/blob/master/pr2_robot/config/output.yaml)  
8. Submit a link to your GitHub repo for the project or the Python code for your perception pipeline and your output `.yaml` files (3 `.yaml` files, one for each test world).  You must have correctly identified 100% of objects from `pick_list_1.yaml` for `test1.world`, 80% of items from `pick_list_2.yaml` for `test2.world` and 75% of items from `pick_list_3.yaml` in `test3.world`.
9. Congratulations!  Your Done!

# Extra Challenges: Complete the Pick & Place
7. To create a collision map, publish a point cloud to the `/pr2/3d_map/points` topic and make sure you change the `point_cloud_topic` to `/pr2/3d_map/points` in `sensors.yaml` in the `/pr2_robot/config/` directory. This topic is read by Moveit!, which uses this point cloud input to generate a collision map, allowing the robot to plan its trajectory.  Keep in mind that later when you go to pick up an object, you must first remove it from this point cloud so it is removed from the collision map!
8. Rotate the robot to generate collision map of table sides. This can be accomplished by publishing joint angle value(in radians) to `/pr2/world_joint_controller/command`
9. Rotate the robot back to its original state.
10. Create a ROS Client for the “pick_place_routine” rosservice.  In the required steps above, you already created the messages you need to use this service. Checkout the [PickPlace.srv](https://github.com/udacity/RoboND-Perception-Project/tree/master/pr2_robot/srv) file to find out what arguments you must pass to this service.
11. If everything was done correctly, when you pass the appropriate messages to the `pick_place_routine` service, the selected arm will perform pick and place operation and display trajectory in the RViz window
12. Place all the objects from your pick list in their respective dropoff box and you have completed the challenge!
13. Looking for a bigger challenge?  Load up the `challenge.world` scenario and see if you can get your perception pipeline working there!

---
### Pipeline implemented
The implemented steps are in the file [project_template.py](/pr2_robot/scripts/project_template.py)
#### Pipeline for filtering and RANSAC plane fitting implemented.
##### 1. Voxel Grid Downsampling 
A voxel grid filter allows you to downsample the data by taking a spatial average of the points in the cloud confined by each voxel. This is because RGB-D cameras provide feature rich and particularly dense point clouds, meaning, more points are packed in per unit volume than, for example, a Lidar point cloud.  `Leaf=0.004`, This means that voxel is 0.004 cubic meter in volume.
Note: Voxel size (or leaf size)  are in meters.

![Alt text](/images/voxel_grid.png)

##### 2. Statistical Outlier Filtering
it is a filtering technique used to remove such outliers (noise due to external factors) is to perform a statistical analysis in the neighborhood of each point, and remove those points which do not meet a certain criteria

##### 3. PassThrough filter
We use axis_min and axis_max pairs to  isolate a region of interest containing only the table and the objects on the table, in this case I filtered 2 axis:
* y: axis_min = -0.5 and axis_max = 0.5
* z: axis_min = 0.6 and axis_max = 1.1
Note: I included `y` axis because it was classifing other objects in the region which only matched with `z` axis. 

![Alt text](/images/pass_through_filter.png)

##### 4.RANSAC Plane Fitting
It is [Random Sample consensus](https://en.wikipedia.org/wiki/Random_sample_consensus) It is used in order to remove the table itself from the scene. The RANSAC algorithm assumes that all of the data in a dataset is composed of both inliers and outliers, where inliers can be defined by a particular model with a specific set of parameters, while outliers do not fit that model and hence can be discarded

##### 5.Extracting Indices - Inliers and Outliers
Once we have determined where the table is located, we can separate point clouds: 
```
extracted_inliers = cloud_filtered.extract(inliers, negative=False) #Table
extracted_outliers = cloud_filtered.extract(inliers, negative=True) #Other objects
```

#### 2. Pipeline including clustering for segmentation implemented.  
##### 1. Euclidean Clustering [DBSCAN](https://en.wikipedia.org/wiki/DBSCAN)
 It is using a PCL library function called EuclideanClusterExtraction() to perform a DBSCAN cluster search for 3D point cloud.
 ![Alt text](/images/clustering_pcl.png)
 
DBSCAN stands for Density-Based Spatial Clustering of Applications with Noise. The DBSCAN algorithm creates clusters by grouping data points that are within some threshold distance `d` from the nearest other point in the data.
it is very useful for the followig scenarios:
* There is an unknown number of clusters present in your data but you know something about the characteristics you expect clusters to have. 
* The data contain outliers that you want to exclude from clusters.
* No need to define a termination / convergence criteria for this algorithm.

In this lesson we also evaluated [k-means clustering](https://en.wikipedia.org/wiki/K-means_clustering), but it was not used because we need to know the number of clusters present.

##### 2. Create Cluster-Mask Point Cloud to visualize each cluster separately
It separates by coloer the shapes obtained. It is used for cluster visualization.

 ![Alt text](/images/cluster-visualization.png)

#### 3. Features extracted and SVM trained.  Object recognition implemented.
##### 3.1 Extract features
* It is using HSV color model
* It is collection histograms as extracted features, ant the result will be normalized. 

##### 3.2 Train data with SVM.
###### 3.2.1 first there was data captured with [capture_features.py](/sensor_stick/scripts/capture_features.py). There was defined the following models:
```
models = [ \
        'biscuits',
        'soap',
        'soap2',
        'book',
        'glue',
        'sticky_notes',
        'eraser',
        'snacks']
```


###### 3.2.2 Train the Support vector machine - [SVM](https://en.wikipedia.org/wiki/Support_vector_machine). It is using [scikit-learn](http://scikit-learn.org/stable/modules/svm.html) library. I tested with the different kernel and I kept `clf = svm.SVC(kernel='linear')` I checked [Kernel functions](http://scikit-learn.org/stable/modules/svm.html#svm-kernels)
The result of this model is the file: 
[perception-model.sav](/models/perception-model.sav)

and the result of this file: 
![Alt text](/images/svm.png)



### Pick and Place Setup

#### 1. For all three tabletop setups (`test*.world`), perform object recognition, then read in respective pick list (`pick_list_*.yaml`). Next construct the messages that would comprise a valid `PickPlace` request output them to `.yaml` format.

these are the final results about [output_1.yaml](/output/output_1.yaml), [output_2.yaml](/output/output_2.yaml), and [output_3.yaml](/output/output_3.yaml):
* ![Alt text](/images/output_1.png)
* ![Alt text](/images/output_2.png)
* ![Alt text](/images/output_3.png)



And here's another image! 
![demo-2](https://user-images.githubusercontent.com/20687560/28748286-9f65680e-7468-11e7-83dc-f1a32380b89c.png)

Spend some time at the end to discuss your code, what techniques you used, what worked and why, where the implementation might fail and how you might improve it if you were going to pursue this project further.  



