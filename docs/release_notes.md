<!--
Copyright (c) 2020 Boston Dynamics, Inc.  All rights reserved.

Downloading, reproducing, distributing or otherwise using the SDK Software
is subject to the terms and conditions of the Boston Dynamics Software
Development Kit License (20191101-BDSDK-SL).
-->

# [Spot SDK](../README.md) > Release Notes

## 2.0.0

### New Features

* **Autonomous Navigation APIs** The APIs used by Autowalk are now accessible to developers.
  * Overall conceptual documents ([link](concepts/autonomy/README.md)):
  * **GraphNavService**: Upload and download maps of the environment, update localizations to that map, and command the robot to autonomously navigate to a location in the map.  Example usage can be found in graph_nav_command_line.py.  Examples of interpreting the map data can be found in graph_nav_view_map
  * **GraphNavRecordingService**: Record maps while the robot walks around. Example usage found in recording_command_line.py
  * **MissionService**: Load and play autonomous missions.  Example mission creation is shown in mission_recorder.py with upload and playback usage shown in replay_mission.py
  * **RemoteMissionService**: A new method for handling mission callbacks, where the mission can trigger user code via an rpc.  For building your own callbacks, see the examples in remote_mission_service.
* **SpotCAM API**. Control and query all hardware features of the SpotCAM.  For examples of using each service, see the spot cam command line example ([link](../python/examples/spot_cam/README.md)).
  * **CompositorService** and **StreamQualityService**: change the layout and quality of the webrtc stream.
  * **PtzService**: Direct PTZ cameras to desired poses.
  * **LightingService**: Control the individual brightness of the illuminator LEDs.
  * **MediaLogService**: Save and retrieve high-resolution images to and from the internal USB drive for later processing.
  * **AudioService**: Upload and play sounds over the SpotCAM speakers.
  * **NetworkService**: Adjust networking settings.
  * **HealthService**, **VersionService**, **PowerService**: Query the status of the hardware and software, and power components on and off.
* **Payload API integration**. Payloads with a compute component can self-register with Spot and API services. ([link](payload/configuring_payload_software.md))
  * **DirectoryRegistrationService**: Allows end users to register new gRPC services running on a payload into the robots service directory.  This allows for communicating through the robot’s proxy to the service from off-robot, and for registering mission callbacks for integrations with Autowalk missions.  See directory_modification.py for example usage.
  * **PayloadRegistrationService**: Payloads can now register themselves with the robot, providing their properties and awaiting authorization from a robot administrator.  See the self_registration and payloads examples for how to register your own payload.
* **Environmental APIs**. Learn more about how Spot is perceiving the world around it.
  * **WorldObjectService**: Request details of any objects that has been detected in the world, and add your own detections.  Example usage can be found in mutate_world_objects.py, fiducial_follow.py, and add_image_coordinates.py.
  * **LocalGridService**: Request maps of the area around the robot, including terrain height and obstacle classification.  Example usage shown by the bosdyn.client command line utility and the basic_streaming_visualizer.py example.
  * **depth_in_visual_frame** image sources. These depth map images have the same dimension, extrinsics, and intrinsics as the grayscale image sources, which can help with pixel-depth correspondence issues.

### Bug Fixes and Improvements

* **Expanded and improved documentation.** 
  * Python QuickStart has been revamped to streamline getting up and running on the SDK.
  * Conceptual documentation has been added to explain key ideas on how to develop for Spot. 
  * Payload developers guide has been added. 
  * Generated documents of the API protocol have also been added.
* **Improved performance over poor communication links.**
  * Reduced API request overhead by several hundred bytes/request.
  * TimeSync estimator more resilient to outlier latencies and temporary network outages.
* **Additional robot state is exposed:**
  * PowerState: Overall charge percentage and estimated runtime.
  * KinematicState: Body velocities are now available in KinematicState.
  * RobotState: Foot contact state state (in contact vs. not in contact).
* **Clients can specify additional advanced locomotion options:**
  * Can now disable various low-level locomotion defaults for special situations and terrain (stair tracking, pitch limiting, cliff avoidance).
  * Body rotation can be specified as an offset to nominal or to horizontal.
* **Consistent Frame usage across API.** See more details in the Breaking Changes section.
* **bosdyn.client command line tool improvements.**
  * Downloading of depth images supported. Depth maps will be written to PGM files.
  * Directory listing has improved formatting.

### Breaking changes

Version 2.0 contains several breaking changes.  While some clients and programs written for the version 1.* SDK may still work, expect some updates to be necessary for most programs.

#### Frame handling

* Documentation of frames on Spot ([link](concepts/geometry_and_frames.md)): 
* The Frame message (geometry.proto) and FrameType have been deprecated, and the frame is now described as a string throughout the API.
* When receiving data from the robot (robot state, images, grid maps, world objects, etc.), the data will come with a string describing the frame it is represented in, but also a FrameTreeSnapshot message which describes how to transform the data into other frames.  
* Use the helpers in frame_helpers.py (in particular get_a_tform_b) to compute appropriate transforms for your use case.  See frame_trajectory_command.py for an example of using transforms to command the robot.
* Code written for version 1 will need to update to the following new convention:

| Version 1 frame enum | Version 2 frame string | frame_helpers.py constant |
|----------------------|------------------------|---------------------------|
| FRAME_KO             | “odom”                 | ODOM_FRAME_NAME           |
| FRAME_VO             | “vision”               | VISION_FRAME_NAME         |
| FRAME_BODY           | “body”                 | BODY_FRAME_NAME           |

#### New Exceptions

* New RpcError exceptions can be raised during rpc calls. If you were already catching RpcErrors, you will catch these.  If you were catching individual subclasses, be aware of these new ones.
  * PermissionDeniedError
  * ResponseTooLargeError
  * NotFoundError
  * TransientFailureError
* There are some new exceptions that can be thrown due to errors with the request before any rpc is made.  They generally indicate programmer error, so depending on your use case it may be acceptable to not catch them to find bugs in your program.  If it is important to catch all exceptions, be aware that these exist, and all inherit from bosdyn.client.Error
  * TimeSyncRequired
  * NoSuchLease
  * LeaseNotOwnedByWallet
* When creating clients or channels from a Robot object, a new class of exceptions inheriting from RobotError may be raised.  NonexistentAuthorityError is no longer thrown, but other RpcErrors may be raised.
  * UnregisteredServiceError
  * UnregisteredServiceNameError
  * UnregisteredServiceTypeError
* Robot command client will throw a new error if a frame is specified that the robot does not recognize.
  * UnknownFrameError

#### Moved or Renamed

* Trajectories must now specify the frame name in the parent message instead of the trajectory itself.
* Trajectory commands can no longer be specified in a body frame since the output behavior can be ambiguous.
* Robot command messages were split into different proto files (basic_command, full_body_command), which will change import/include paths.
* ‘vel’ field in SE3TrajectoryPoint renamed to ‘velocity’ (trajectory.proto)
* Updates in Payload.proto
  * LabelPrefix field was changed from String to Repeated String
  * body_T_payload renamed to body_tform_payload
  * mount_T_payload renamed to mount_tform_payload

#### Removed

* All Frame messages have been replaced by frame strings where applicable.
* AddLogAnnotationResponse does not have a status field anymore, errors are encoded in the message header information
* ko_tform_body, vo_tform_body, and ground_plane_rt_ko in Kinematic state have been replaced with the transforms_snapshot.
* The SampleCommon message for image captures has been replaced by acquisition time and a FrameTreeSnapshot.

#### Miscellaneous

* bosdyn-client has added a dependency on numpy
* Autowalk missions and maps recorded with version 1.1 are not compatible with version 2.0

### Known issues

* If you delete an object from the world object service, there is a chance that a ListWorldObjects call immediately afterwards may still include that object.
  * Workaround: wait a short time before expecting the object to be gone.
* If you register a new service with the robot, calling robot.ensure_client to create a client for that service may result in a UnregisteredServiceNameError. 
  * Workaround: call robot.sync_with_directory() before robot.ensure_client()
* SE2VelocityLimits require care.  The proto comment states that "if set, limits the min/max velocity," implying that one should not set values for any directions one does not want limited. However, if any of the numeric fields are not set in the message, they will be interpreted as 0. For example, if angular is not set but linear is, then the message will be incorrectly interpreted as having an angular limit of 0 and the robot will fail to rotate (obviously not the intent). Similarly, if the user only sets say the 'x' field of linear, then 'y' will be incorrectly limited to 0 as well.
  * Workaround: Correct usage of the SE2VelocityLimit message requires the user to fully fill out all the fields, setting unlimited values to a large number, say 1e6.
* LogAnnotationClient does not include async versions of its rpcs.
  * Workaround: If you need to call these in an async manner, call them on a separate thread.

### Sample Code
* **directory** ([link](../python/examples/directory/README.md))
  * Register, update, and unregister a service
* **payloads** ([link](../python/examples/payloads/README.md))
  * Renamed from get_payload
  * Expanded to show payload version handling
* **self_registration** ([link](../python/examples/self_registration/README.md))
  * Example showing how to set up a payload that registers itself with the robot, hosts a service, and registers its service with the robot.
* **add_image_coordinates_to_scene** ([link](../python/examples/add_image_coordinates_to_scene/README.md))
  * Example using the API demonstrating adding image coordinates to the world object service
* **estop** (updated) ([link](../python/examples/estop/README.md))
  * New EstopNoGui as a command-line addition to the GUI version of the Estop example 
* **get_depth_plus_visual_image** ([link](../python/examples/get_depth_plus_visual_image/README.md))
  * Example demonstrates how to use the new depth_in_visual_frame image sources to visualize depth in a fisheye image.
* **get_mission_state** ([link](../python/examples/get_mission_state/README.md))
  * Example program demonstrates how to retrieve information about the state of the currently-running mission.
* **frame_trajectory_command** ([link](../python/examples/frame_trajectory_command/README.md))
  * Example program shows how to retrieve Spot's location in both the visual and odometry frames. Using these frames, the program shows how to build and execute a command to move Spot to that location plus 1.0 in the x axis.
* **get_robot_state_async** ([link](../python/examples/get_robot_state_async/README.md))
  * Example demonstrates 3 different methods for working with Spot asynchronous functions.
* **get_world_objects** ([link](../python/examples/get_world_objects/README.md))
  * Example demonstrate how to use the world object service to list the objects Spot can detect, and filter these lists for specific objects or objects after a certain time stamp.
* **graph_nav_command_line** ([link](../python/examples/graph_nav_command_line/README.md))
  * Command line interface for graph nav with options to download/upload a map and to navigate a map.
* **graph_nav_view_map** ([link](../python/examples/graph_nav_view_map/README.md))
  * Example shows how to load and view a graph nav map.
* **mission_recorder** ([link](../python/examples/mission_recorder/README.md))
  * Example with an interface for operating Spot with your keyboard, recording a mission, and saving it.
* **remote_mission_service** ([link](../python/examples/remote_mission_service/README.md))
  * Run a gRPC server that implements the RemoteMissionService service definition
  * Connect a RemoteClient directly to that server.
  * Build a mission that talks to that server.
* **replay_mission** ([link](../python/examples/replay_mission/README.md))
  * Example on how to replay a mission via the API
* **spot_cam** ([link](../python/examples/spot_cam/README.md))
  * Examples to demonstrate how to interact with the Spot CAM.
* **visualizer** ([link](../python/examples/visualizer/README.md))
  * Example to visualize Spot's perception scene in a consistent coordinate frame. 
* **world_object_mutations** ([link](../python/examples/world_object_mutations/README.md))
  * Examples to demonstrate how to use the world object service to add, change, and delete world objects.
* **xbox_controller** (updated) ([link](../python/examples/xbox_controller/README.md))
  * Added support for Windows driver

## 1.1.2

### New features

* **Missions API Beta and Callbacks** When an Autowalk Mission is created with a "Callback" event, a client can detect that change using a beta version of the Mission API, then tell the robot to continue or abort the Mission. Example uses include a sensor payload that detects Callback events and captures sensor information before advancing the mission, and a desktop UI which waits for the user to push a button before advancing the mission.

* **External Forces Beta**. Mobility commands can specify how to handle external forces. Examples of external forces could include an object that Spot is towing, or an object Spot is pushing. The default behavior is to do nothing, but clients can specify whether Spot should estimate external forces and compensate for it, or explicitly specify what the external forces are.

### Bug fixes and improvements

* **Depth image** extrinsics are fixed. In earlier 1.1.x releases, the extrinsics were incorrectly the same as the fisheye cameras.

* **App Token expiration logging.**. The Python SDK object logs if the app token will expire in the next 30 days. New tokens can be requested at support@bostondynamics.com.

### Sample Code

* **mission_question_answerer** demonstrates how to build a client which responds to Mission Callbacks.

## 1.1.1

1.1.1 has no SDK-related changes.

## 1.1.0

The 1.1.0 SDK software is published under a new software license which can be found in the LICENSE file at the top of the SDK directory.

### New features

* **ImageService** exposes depth maps from on-board stereo cameras as additional image sources. Image Sources specify an ImageType to differentiate depth maps from grayscale images.

* **PayloadService** is a new service which lists all known payloads on the robot.

* **SpotCheckService** is a new service which runs actuator and camera calibration.

* **E-Stop soft timeouts.** In prior software release, E-Stop endpoints which stopped checking in would result in the power to the robot's motors being cut immediatly. Now the E-Stop endpoint can be configured so Spot will attempt to sit followed by cutting power. The timeout parameter for an E-Stop endpoint specifies when the sitting behavior starts, and the cut_power_timeout parameter specifies when the power will cut off.

* **TerrainParams** can be added to MobilityParams to provide hints about the terrain that Spot will walk on. The coefficient of friction of the ground can be specified. Whether the terrain is grated metal can also be specified.

* **Log Annotations** a new type of log: Blob for large binary data.

### Sample code

The sample code directory structure has changed to directory-per-example under python/examples. Each example includes a requirements.txt file for specifying dependencies.

* **estop** is a desktop GUI which creates an E-Stop endpoint for a robot. This example demonstrates how to use the E-Stop endpoint system, and is a useful utility on its own.

* **follow_fiducial** is an example where Spot will follow an AprilTag fiducial that it can see in its on-board cameras. This demonstrates how to use camera extrinsics and intrinsics to convert pixels to world coordinates, and how to use the trajectory commands.

* **get-image** is a simple example to retrieve images from Spot and save them to disk. It shows how to use the basics of the image service.

* **get-payload** is a simple example which lists all of the payloads on the robot. It shows how to use the payload service.

* **get_robot_state** is a simple example to retrieve robot state.

* **hello_spot** is carried over from prior SDK releases. It is an introductory tutorial showing basic use of the Spot SDK.

* **logging** demonstrates how to add custom annotations to Spot’s log files.

* **spot_check** demonstrates how to use SpotCheck - Spot’s in-the-field calibration behavior..

* **spot_light** is a demo where Spot rotates its body while standing to try to face a flashlight seen in its front cameras.

* **spot_tensorflow_detector** demonstrates how to integrate the Spot SDK with Tensorflow for object classification from images.

* **time_sync** is a simple example demonstrating how to use the TimeSync service.

* **wasd** is carried over from prior SDK releases. It is an interactive program which uses keyboard control for the robot, and demonstrates how to use a variety of commands.

* **xbox_controller** demonstrates how to specify more advanced options for mobility commands.

### Bug fixes and Improvements

* Too many invalid login attempts will lock a user out from being able to authenticate for a temporary period to prevent brute-force attacks. GetAuthTokenResponse indicates this state with a STATUS_TEMPORARILY_LOCKED_OUT.

* Elliptic Curve (ECDSA) cryptography used for user tokens - reducing the size of RPC requests by several hundred bytes.

* gRPC exceptions are correctly handled in Python 3.

* Command-line tool handles unicode robot nicknames correctly.

* Command-line tool supports retrieving robot model information (URDF and object files)

* Command-line tool supports retrieving multiple images at once.

* “Strict Version” support for software version.

* App Token paths which include “~” will automatically expand, rather than fail.

* Mobility Commands which have MobilityParams.vel_limit with only a min or max velocity are correctly handled. In prior releases, these commands would result in no movement at all.

* Mobility Commands which have BodyControlParams.base_offset_rt_footprint.reference_time in the past result in a STATUS_EXPIRED response rather than a STATUS_INVALID_REQUEST.

* SE2VelocityCommands with unset slew_rate_limit are correctly handled. In prior releases, an unset slew_rate_limit would result in the robot walking in place.

### Deprecations and breaking changes

* Removed support for Python 2.7. Only Python 3.6+ is supported due to upcoming Python 2 End-of-Life. Windows 10, macOS, and Ubuntu LTS are still supported.

* The HINT_PACE LocomotionHint is no longer supported due to physical stability issues. Any commands which specify a HINT_PACE LocomotionHint will be treated like a HINT_JOG LocomotionHint is specified.

* HINT_AUTO_TROT and HINT_AUTO_AMBLE are deprecated as names - use HINT_SPEED_SELECT_TROT and HINT_SPEED_SELECT_AMBLE respectively going forward.

* Protocol Buffer locations changed to split services from messages. Any direct imports of protocol buffers by client applications will need to change to support the 1.1.0 version changes.

## 1.0.1

* Improved documentation on SDK installation.

* Clearer Python dependency requirements.

* RobotId service exposes computer serial number in addition to robot serial number.

* wasd image capture works in Python 3.

* Fixed timing bugs in power service.

## 1.0.0

Initial release of the SDK.
