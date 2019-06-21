# Azure Kinect ROS Driver Usage

The Azure Kinect ROS Driver node exposes Azure Kinect sensor streams to ROS. 

## Setup and Building

Please see the [building guide](building.md).

### Parameters

The Azure Kinect ROS Driver node accepts a number of [ROS Parameters](http://wiki.ros.org/Parameter%20Server) to configure the Azure Kinect sensor. Since the node uses the ROS parameter server, these parameters can be set in the usual ROS ways (on the command line, in a launch file, through the parameter server, etc..).

The node accepts the following parameters:

- `sensor_sn` (string) : No default value. The serial number of the Azure Kinect device. If this parameter is not specified, the node will auto-select the first Azure Kinect device.
- `depth_enabled` (bool) : Default to '`true`'. Controls if the depth camera will be turned on.
- `depth_mode` (string) : Defaults to '`NFOV_UNBINNED`'. This string selects the depth camera operating mode. More details on the various depth camera modes can be found in the Azure Kinect documentation. Valid options are '`NFOV_2X2BINNED`', '`NFOV_UNBINNED`', '`WFOV_2X2BINNED`', '`WFOV_UNBINNED`', '`PASSIVE_IR`'.
- `color_enabled` (bool) : Defaults to '`false`'. Controls if the color camera will be turned on.
- `color_resolution` (string) : Defaults to '`720P`'. This string selects the color camera resolution. The selected color camera resolution will affect the overlap between the depth and color camera field-of-view, which will impact the quality / resolution of the depth-to-color and color-to-depth point clouds. More details on the various color-camera resolutions and their aspect ratios can be found in the Azure Kinect documentation. Va;od options are '`720P`', '`1080P`', '`1440P`', '`1536P`', '`2160P`', '`3072P`'.
- `fps` (int) : Defaults to `5`. This parameter controls the FPS of the color and depth cameras. The cameras cannot operate at different frame rates. Valid options are `5`, `15`, `30`. Note that some FPS values are not compatible with high color camera resolutions or depth camera resolutions. For more information, see the Azure Kinect documentation.
- `point_cloud` (bool) : Defaults to `true`. If this parameter is set to `true`, the node will generate a sensor_msgs::PointCloud2 message from the depth camera data. This requires that the `depth_enabled` parameter be `true`.
- `rgb_point_cloud` (bool) : Defaults to `false`. If this parameter is set to `true`, the node will generate a sensor_msgs::PointCloud2 message from the depth camera data and colorize it using the color camera data. This requires that the `point_cloud` parameter be `true`, and the `color_enabled` parameter be `true`.

#### Parameter Restrictions

Some parameters are incompatible with each other. The ROS node attempts to detect incompatible parameters and provide a runtime error to roserr: however, not all potential incompatibilities have been accounted for. In these instances, the Azure Kinect SDK may throw an exception.

Some example incompatibilities are provided here:
- The `rgb_point_cloud` parameter requires that both the depth and color camera be enabled. This means that `depth_enabled` and `color_enabled` must both be `true`, `depth_mode`, `color_resolution`, and `fps` must be valid.
- Some combinations of depth mode / color resolution / FPS are incompatible. The Azure Kinect SDK will generate an exception and print a log message when this situation is detected. The ROS node will then exit.
- Enabling `rgb_point_cloud` will restrict the resolution of the emitted point cloud to the overlapping region between the depth and color cameras. This region is significantly smaller than the normal field-of-view of the depth camera.

## Topics

The node emits a variety of topics into its namespace. 

- `points2` (`sensor_msgs::PointCloud2`) : The point cloud generated by the Azure Kinect SDK from the depth camera data. If the `rgp_point_cloud` option is set, the points in the cloud will be colorized using information from the color camera. 
- `rgb/image_raw` (`sensor_msgs::Image`) : The raw image from the color camera, in BGRA format. Note that this image is **not** undistorted. The image can be undistorted using the [image_proc package](http://wiki.ros.org/image_proc).
- `rgb/camera_info` (`sensor_msgs::CameraInfo`) : Calibration information for the color camera, converted from the Azure Kinect format. The Azure Kinect uses the `rational_polynomial` distortion model.
- `depth/image_raw` (`sensor_msgs::Image`) : The raw image from the depth camera, in 32FC1 format. This differs from previous ROS Kinect drivers, which emitted data in the Kinect-native MONO16 format in units of millimeters. Note that this image is **not** undistorted. The image can be undistorted using the [image_proc package](http://wiki.ros.org/image_proc).
- `depth/camera_info` (`sensor_msgs::CameraInfo`) : Calibration information for the depth camera, converted from the Azure Kinect format. The Azure Kinect uses the `rational_polynomial` distortion model.
- `depth_to_rgb/image_raw` (`sensor_msgs::Image`) :  The depth image, transformed into the color camera co-ordinate space by the Azure Kinect SDK. This image has been resized to match the color image resolution. Note that since the depth image is now transformed into the color camera co-ordinate space, some depth information may have been discarded if it was not visible to the depth camera.
- `depth_to_rgb/camera_info` (`sensor_msgs::CameraInfo`) : A copy of the color camera calibration which has been modified to match the depth co-ordinate frame. The depth camera image provided by `depth_to_rgb/image_raw` is distorted in the same way as the color camera image provided by `rgb/image_raw`. Correctly undistorting this image require the use of the color camera calibration, which is provided on this topic.
- `rgb_to_depth/image_raw` (`sensor_msgs::Image`) : The color image, transformed into the depth camera co-ordinate space by the Azure Kinect SDK. This image has been resized to match the depth image resolution, however depth pixels that are not visible to the color camera will return with a black color. 
- `rgb_to_depth/camera_info` (`sensor_msgs::CameraInfo`) : A copy of the depth camera calibration which has been modified to match the color camera co-ordinate frame. The color camera image provided by `rgb_to_depth/image_raw` is distorted in the same way as the depth camera image provided by `depth/image_raw`. Correctly undistorting this image require the use of the depth camera calibration, which is provided on this topic.
- `imu` (`sensor_msgs::Imu`) : The intrinsics-corrected IMU sensor stream, provided by the Azure Kinect Sensor SDK. The sensor SDK automatically corrects for IMU intrinsics before sensor data is emitted.

## Azure Kinect Calibration

Unlike previous ROS Kinect drivers, the Azure Kinect ROS Driver provides calibration data from the sensor, decoded from the Azure Kinect native data format. This calibration information is used to populate several important pieces of ROS calibration information.

- tf2 : The relative offsets of the cameras and IMU are loaded from the Azure Kinect extrinsics calibration information. The various co-ordinate frames published by the node to TF2 are based on the calibration data. Please note that the calibration information only provides the relative positions of the IMU, color and depth cameras. It does not provide the offset between the sensors and the mechanical housing of the camera. The transform between the `camera_base` frame and `depth_camera_link` frame are based on the mechanical specifications of the Azure Kinect, and are not corrected by calibration.
- sensor_msgs::CameraInfo : Intrinsics calibration data for the cameras is converted into a ROS-compatible format. Fully populated CameraInfo messages are published for both the depth and color cameras, allowing ROS to undistort the images using image_proc and project them into point clouds using depth_image_proc. Using the point cloud functions in this node will provide GPU-accelerated results, but the same quality point clouds can be produced using standard ROS tools.