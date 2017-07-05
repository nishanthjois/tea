# Team TEA - Udacity / Didi entry
This is the code entry that got `0.3824329` score in round two of the Udacity / Didi competition.

### Dependencies
This obviously requires ROS. Please refer to the [ROS Documentation](http://wiki.ros.org/ROS/Tutorials/BuildingPackages) on how to build catkin packages.

Other dependencies:
* numpy 1.13.0
* [ROS Velodyne driver](https://bitbucket.org/DataspeedInc/ros_binaries)
* PCL 1.8 (may work with 1.7)
* [python-pcl bindings](https://github.com/duburlan/python-pcl)
* Keras 2.0.5
* Tensorflow 1.2
* HDF5

## Generating tracklets

The `tracklets` directory contains two convenience scripts for generating tracklets for car and pedestrian bags. As per the rules of the competition we used one single ros node which handles car and pedestrian bags via different command-line arguments: **for cars bags we used exactly the same parameters for all car bags**, as can be verified by looking at the XML comment in each tracklet:

```
<?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
<!-- Generated with -sm ../scripts/torbusnet/lidarnet-car-r3-x13-d40-seg-rings_12_28-sectors_32-epoch37-val_class_acc0.9992.hdf5 -lm ../scripts/torbusnet/lidarnet-car-r2-fs-di-yawbalance-loc-rings_12_28-epoch10-val_loss0.0333.hdf5 -di -rfp -lpt 3 -bsw 1.2 -zc 0.06 -yc 0.075 -bsh 1.1 -b /home/antor/didi-ext/didi-data/release3/Data-points/testing/ford02.bag -->
<!DOCTYPE boost_serialization>
<boost_serialization signature="serialization::archive" version="9">
<tracklets class_id="0" tracking_level="0" version="0">
...
``` 

Before running either `cars.sh` or `ped.sh` make sure to:
* edit the full path of the location of the test bags, i.e. edit this path appropriately:

```
BAGPATH=/home/antor/didi-ext/didi-data/release3/Data-points/testing
``` 
* `roscore` has been started
* the pedestrian bag has the `/velodyne_points` topic (the original bag provided by Udacity did NOT contain it)

Both the `cars.sh` and `ped.sh` script just start our `ros_node.py` ros node with paths to the two model pairs we used (one model pair for the pedestrian and another for cars), as well as extra command-line arguments that control configurable hyper-parameters for the whole pipeline.

### Car tracklets

Car tracklets are generated by invoking the `car.sh` script as follows:

```
$ ./cars.sh \
../scripts/torbusnet/lidarnet-car-r3-x13-d40-seg-rings_12_28-sectors_32-epoch37-val_class_acc0.9992.hdf5 \
../scripts/torbusnet/lidarnet-car-r2-fs-di-yawbalance-loc-rings_12_28-epoch10-val_loss0.0333.hdf5 \
"-bsw 1.2 -zc 0.06 -yc 0.075 -bsh 1.1"
```

The tracklets for the final submission are in the `scripts/tracklets` directory and they contain XML comments with the exact arguments we used in the final submission. The organization can verify we used the same exact parameters for all car bags (for pedestrian bags this is a non-issue because there is only one bag). **Should other teams be allowed to fine-tune command-line arguments on a per-bag basis we would request Udacity our chance to do so to compete on equal conditions**. We strongly believe the spirit of the competition was to provide a general solution and not one that attempted to overfit on a per-bag basis.

### Pedestrian tracklet

Generate the pedestrian by invoking `ped.sh` script as follows:

```
$ ./ped.sh \
../scripts/torbusnet/lidarnet-ped-r3-x135-d40-seg-rings_12_28-sectors_32-epoch14-val_class_acc0.9990.hdf5 \
../scripts/torbusnet/lidarnet-ped-r3-d45-loc-rings_12_28-epoch01-val_loss0.0044.hdf5 \
"-yc 0.4 -pc -0.5 -zc -0.1"
```

## Performance

### Qualitative analysis

[Here's a fragment](https://youtu.be/HlxnaiGQzmE) of the visualization of one of the car bags. Note that is an early video and the final tracklets accomplish much better peformance. 

## Real-time performance

The pipeline makes predictions in less that 100 msecs:

![whole pipeline real time performance](https://s3.amazonaws.com/team-tea-udacitydidi/team-tea-pipeline-peformance.png)

This has been tested in an Intel(R) Core(TM) i5-4690K CPU @ 3.50GHz and Geforce GTX 1080 Ti.

## Pipeline explanation

The pipeline roughly consists of the following steps:

1. **Lidar segmentation** A deep recurrent neural network that takes the lidar point cloud processed in a special way and segments lidar points outputting the probably of each point being an obstacle or not. We've coined this part `lidarnet-segmenter`. 

We've used a novel approach where instead of taking the lidar point cloud and create image-like features, we take the readings of the lidar as 64 1-dimensional signals, 2 signals (distance and intensity) for each of the 32 lidar rings:

![lidar visualized](http://www.mdpi.com/remotesensing/remotesensing-07-10480/article_deploy/html/images/remotesensing-07-10480-g001-1024.png)

For this top-view of the lidar cloud:

![lidar top view](https://s3.amazonaws.com/team-tea-udacitydidi/lidar-topview.png)

...where the obstacle has a blue bounding box, we generate the following signals:

![lidar rings](https://s3.amazonaws.com/team-tea-udacitydidi/lidar-rings.png)

the X axis goes from -Pi to Pi, and the Y axis is the distance for each ring. In the image above each lidar ring (the HDL-32e has 32 but we are only using some of them) is painted with a different color for clarity. There's an equivalent signal for the lidar intensities but is not shown here. We've just painted the angle of the bounding box as seen by the lidar as vertical dotted lines. 

We trained a recurrent neural network to predict each point of each ring that belongs to an obstacle:

![lidarnet segmenter](https://s3.amazonaws.com/team-tea-udacitydidi/segmenter.gif)

The left image is the ground truth where the obstacle is in red, and the right image shows the predictions (note this video is from an early test and is not representative of the final model we used).

2. **Clustering and filtering** We use simple false positive rejections by rejecting points that are too far away from the next prediction location.

3. **Lidar points localization** If/when we have a points that are segmented as obstacle points, we infer the bounding box centroid, size and yaw using a [Pointnet](https://arxiv.org/abs/1612.00593) inspired point cloud regressor. We've coined this part `lidarnet-localizer`. 

4. **Unscented kalman filter processing** We fuse radar measurements and *lidarnet provided* bounding box position to make predictions upon camera messages.

5. **Post-processing** Because we did not have time to polish the round 2 ground truth and had to train the `lidarnet-segmenter` using data from release 2 (round 1) which had even fewer vehicle sizes we adjust bounding box size and small lidar misalignments with contant parameters.

## Future work

The pipeline meets the requirements of the Didi-Udacity competition but it doesnt stop there. The *lidarnet segmenter* is able to detect multiple objects, thanks to the way training works (see `lidarnet.py`). Furthermore, we could use a phased-LSTM instead of GRU to further optimize training and avoid interpolation (and de-interpolation). We also did not have time to fine-tune the unscented kalman filter including a better state model for *yaw*. 
