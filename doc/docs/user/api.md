# <a name="topology">Topology</a>
The topology that needs to be send to Task Planner is composed of nodes and edges. In the future it will be a single graph message, but since firos is not supporting arrays of custom ROS messages it is divided into two ROS messages: nodes and edges.

Nodes.msg

	Header header	# standard ROS header
	nav_msgs/MapMetaData info	# number of cells in x and y directions of the gridmap, the size of the cell
	float64[] x	# x coordinate of the cell centre
	float64[] y	# y coordinate of the cell centre
	float64[] theta # orientation of the node used for annotations
	string[] name	# e.g. vertex_0 or annotation name
	string[] uuid	# unique id of a node
	
Edges.msg

	Header header	# standard ROS header 
	string[] uuid_src	# unique id of a source node of the edge
	string[] uuid_dest	# unique id of a destination node of the edge
	string[] name	# e.g. edge_0_1
	string[] uuid	# unique id of an edge
	
The creation of topology is started by launching the map_server package followed by launching the maptogridmap package which is explained in more details in the following text:
```
terminal 1: roslaunch maptogridmap startmapserver.launch
terminal 2: roslaunch maptogridmap startmaptogridmap.launch
```
You can also start a single launch file which combines these two launch files:
```
roslaunch maptogridmap topology.launch
``` 
## Creation of Nodes in maptogridmap package
Nodes are all free cells of rectangular square size that does not contain any obstacle within it. The obstacles are read from the map PNG or PGM file loaded by calling the map_server node. There are three maps prepared in startmapserver.launch, where MURAPLAST florplan is uncommented and IML lab and ICENT lab are commented out for the later usage.

```
<launch>
<!--MURAPLAST floorplan-->
<node name="map_server" pkg="map_server" type="map_server" args="$(find maptogridmap)/launch/floorplanMP.yaml" respawn="false" >
<param name="frame_id" value="/map" />
</node>


<!--IML lab floorplan	
<node name="map_server" pkg="map_server" type="map_server" args="$(find maptogridmap)/launch/IMLlab.yaml" respawn="false" >
<param name="frame_id" value="/map" />
</node>
-->	
		
<!--ICENT lab floorplan
<node name="map_server" pkg="map_server" type="map_server" args="$(find maptogridmap)/launch/Andamapa.yaml" respawn="false" >
<param name="frame_id" value="/map" />
</node>
-->

<node name="rviz" pkg="rviz" type="rviz" args="-d $(find maptogridmap)/singlerobot.rviz" /> 

</launch>
```

The size of the cell is given by the parameter in startmaptogridmap.launch:
```
<launch>

    <node name="map2gm" pkg="maptogridmap" type="map2gm" >
        <param name="cell_size" type="double" value="2.0" />
        <param name="annotation_file" textfile="$(find maptogridmap)/launch/annotations.ini" />
    </node>

</launch>
```

In this example it is set to 2.0m since the floorplan is quite big. ICENT lab is much smaller so 1.2m cell size gives better results. Values that are presented in context broker are coordinates of the cell center (x,y) or coordinates of the manual annotation (loaded from a file), theta as an orientation of the annotated place in the map (default is 0), a name of the node in the form of "vertex_0" or annotated name, and the node's uuid. The message that is sent through firos can be found here: maptogridmap/msg/Nodes.msg.

### Annotations

Annotations can be loaded from file annotations.ini located in maptogridmap/launch folder, which is put as a parameter textfile inside the startmaptogridmap.launch file. In this example the first three annotations were used in Zagreb demo in Task planner, and P1 is put as an additional example for the IML map:
```
[loadingArea]
# coordinates
point_x = 3.75
point_y = 2.51
theta = 0
distance = 1.6

[unloadingArea]
# coordinates
point_x = 6.4
point_y = 2.51
theta = 0 
distance = 1.4

[waitingArea]
# coordinates
point_x = 6.9
point_y = 4.3
theta = 90
distance = 1.6

[P1]
# coordinates
point_x = 17.96
point_y = 6.57
theta = -90
distance = 1.8
```
The annotations are saved under the variable of type maptogridmap::Annotations inside of the maptogridmap package. All values must be in meters and degrees. From these values it is calculated where the AGV needs to be placed in front of the annotation according to the distance from the annotation and the orientation theta. It changes the values of the computed nodes from gridmap cells so that TP can use this nodes as goals.
```
x[]
  x[0]: 3.75
  x[1]: 6.4
  x[2]: 6.9
  x[3]: 17.96
y[]
  y[0]: 2.51
  y[1]: 2.51
  y[2]: 4.3
  y[3]: 6.57
theta[]
  theta[0]: 0
  theta[1]: 0
  theta[2]: 90
  theta[3]: -90
distance[]
  distance[0]: 1.6
  distance[1]: 1.4
  distance[2]: 1.6
  distance[3]: 1.8
name[]
  name[0]: loadingArea
  name[1]: unloadingArea
  name[2]: waitingArea
  name[3]: P1
```
These four annotations change the coordinates of the cell centre of the grid map (but only free cells) and also change the name to the annotation name, e.g., loadingArea, unloadingArea, etc. The result can be seen in [topic /map/nodes.](#exampleannot)


## Creation of Edges in maptogridmap package
Edges are pairs of neighbor nodes. Neighbors are defined between two nodes which have their centres' coordinates distanced for _cell_size_. The edges are bidirectional, meaning two neighbor nodes n and m forms the edge (n,m) and (m,n) which are identical.
Values of Edges are the source node's uuid, named as _uuid_src_, the destination node's uuid, named as _uuid_dest_, the name of the edge in the form of "edge_0_1" meaning that two nodes with names "vertex_0" and "vertex_1" are connected with the edge, and the edge's uuid. 
The message that is sent through firos can be found here: maptogridmap/msg/Edges.msg

## <a name="writelis">Writing a simple listener explaining the maplistener package</a>

To read the topic in your own package you need to subscribe to it, include the header of the message, and write a message callback. The example is taken from maplistener/src/main.cpp.

* subscribe to a topics /map/nodes and /map/edges 
```
 	nodes_sub = nh_.subscribe("map/nodes",1,&VisualizationPublisherGML::nodesCallback, this);
 	edges_sub = nh_.subscribe("map/edges",1,&VisualizationPublisherGML::edgesCallback, this);

```
* include the header of the message in your header file or in the cpp where you are writting the message callback:
```
#include <maptogridmap/Nodes.h>
#include <maptogridmap/Edges.h>
```
* write a message callback for nodes and edges and save the values in the global variable gmnode that needs to be saved for obtaining the coordinates of edges
```
maptogridmap::Nodes gmnode;
void VisualizationPublisherGML::nodesCallback(const maptogridmap::NodesConstPtr& gmMsg)
{
  graphvs.points.clear();
  geometry_msgs::Point p; 
	gmnode.x.clear();
	gmnode.y.clear();
	gmnode.name.clear();
	gmnode.uuid.clear();
	for (int i=0; i<gmMsg->x.size(); i++){
		p.x=gmMsg->x[i];
		p.y=gmMsg->y[i];
		graphvs.points.push_back(p);
		gmnode.x.push_back(gmMsg->x[i]);
		gmnode.y.push_back(gmMsg->y[i]);
		gmnode.name.push_back(gmMsg->name[i]);
		gmnode.uuid.push_back(gmMsg->uuid[i]);
	}
}
void VisualizationPublisherGML::edgesCallback(const maptogridmap::EdgesConstPtr& gmMsg)
{
  stc.points.clear();
  geometry_msgs::Point p; 
  int foundsrcdest;
	for (int i=0; i<gmMsg->name.size(); i++){
		foundsrcdest=0;
		for (int j=0; j<gmnode.name.size(); j++){
			if (gmnode.uuid[j]==gmMsg->uuid_src[i]){
				p.x=gmnode.x[j];
				p.y=gmnode.y[j];
				stc.points.push_back(p);
				foundsrcdest++;
				if (foundsrcdest==2)
					break;
			}
			if (gmnode.uuid[j]==gmMsg->uuid_dest[i]){
				p.x=gmnode.x[j];
				p.y=gmnode.y[j];
				stc.points.push_back(p);
				foundsrcdest++;
				if (foundsrcdest==2)
					break;
			}
		}
	}
}
```
* add in CMakeLists.txt of your package the line _maptogridmap_ in find_package function otherwise the compiler will complain it can not find the header file:
```
find_package(catkin REQUIRED COMPONENTS
  std_msgs
  nav_msgs
  geometry_msgs
  message_generation
  roscpp
  tf
  maptogridmap
  mapupdates
)
```
* add depend _maptogridmap_ in your package in package.xml:
```
  <build_depend>maptogridmap</build_depend>
  <run_depend>maptogridmap</run_depend>
```
You can test how subscribed topics are visualized in rviz by typing:
```
terminal 1: roslaunch maptogridmap startmapserver.launch
terminal 2: roslaunch maptogridmap startmaptogridmap.launch
terminal 3: rosrun maplistener mapls
```
The nodes are visualized with the marker topic /nodes_markerListener, while the edges are visualized with the marker topic /edges_markerListener.
The package maplistener also subscribes to map updates for which you first need to have localization (AMCL) and laser readings (from simulator Stage) as explained in the Section [Map updates](#mapupdates):
```
terminal 4: roslaunch lam_simulator AndaOmnidriveamcltestZagrebdemo.launch
terminal 5: roslaunch mapupdates startmapupdates.launch
```

# <a name="poswithcov">Pose with covariance</a>

A standard ROS message is used for sending the pose of the AGV:
	
	geometry_msgs.msg.PoseWithCovarianceStamped
	
To be able to send this message, two modules needs to be started: the AMCL package and the sensing_and_perception package. The AMCL package provides localization of the AGV inside a given map using the odometry and laser sensors. The sensing_and_perception package combines the covariance calculated by the AMCL package and the the global pose in the map frame as a result of a combination of odometry and AMCL localization with the laser.
Since this topic will be send through firos to OCB, the unique robot id needs to be set in the lauch file through _args_, e.g. 0, 1, 2, ... In this example launch file send_posewithcovariance.launch you can see that the id of the robot is set to 0:
```
<launch>

     <!--- Run pubPoseWithCovariance node from sensing_and_perception package-->
     <!-- Put args="1" if you are testing the robot with the id number 1 -->
     <node name="publishPoseWithCovariance" pkg="sensing_and_perception" type="pubPoseWithCovariance" output="screen" args="0"/>	

</launch>
```

Example with the MURAPLAST factory floorplan:
```
terminal 1: roslaunch lam_simulator amcl_test_muraplast.launch
terminal 2: roslaunch sensing_and_perception send_posewithcovariance.launch 
```

Example with the ICENT lab floorplan in Zagreb review meeting demo:
```
terminal 1: roslaunch lam_simulator AndaOmnidriveamcltestZagrebdemo.launch
terminal 2: roslaunch sensing_and_perception send_posewithcovariance.launch 
```

Example with the IML lab floorplan:
```
terminal 1: roslaunch lam_simulator IMLamcltest.launch
terminal 2: roslaunch sensing_and_perception send_posewithcovariance.launch 
```
As the result you can echo the topic /robot_0/pose_channel:
```
rostopic echo /robot_0/pose_channel
```
If you want to send this topic through firos, use and adapt the config files in firos_config inside the localization_and_mapping metapackage (or from test/config_files/machine_1) and start firos by typing:
```
terminal 3: rosrun firos core.py 
```

#SLAM

When the map (or we also say the initial map) is not available, the SLAM process needs to be used to create it.

Inside the package lam_simulator there is a launch file (folder _localization_and_mapping/lam_simulator/launch/_). Run:

```
roslaunch roslaunch lam_simulator gmapping_test_muraplast.launch
```


The launch file starts gmappping SLAM algorithm, the simulation arena with the MURAPLAST factory floorplan and RVIZ.
The robot can be moved by moving the mouse while holding the pointer over the robot, or by using the ROS teleoperation package which is used for the real robot:

```
rosrun teleop_twist_keyboard teleop_twist_keyboard.py
```

The mapped environment is visualized in RVIZ.
After you are sattisfied with the built map presented in RVIZ, save it and use it later for AMCL localization by starting the map saver:

```
rosrun map_server map_saver -f mapfile
```

#Preparing the built map to be used for localization and navigation

As a result of SLAM you will obtain in the current folder where you called the map_saver command the mapfile.pgm and mapfile.yaml files.
This is an example:

```
image: mapfile.pgm
resolution: 0.068000
origin: [-25.024000, -25.024000, 0.000000]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.196
```

To simulate this map in Stage you need to do the following. By default map_server and Stage have different reference global map coordinate systems. We choose to have the coordinate system at the lower left corner of the map. To have that first autocrop the mapfile.pgm so that borders are occupied, otherwise map_server and Stage will have different scales since Stage does the autocrop by itself. To do so open the mapfile.pgm in gimp, autocrop and put it in lam_simulator/worlds/elements folder for Stage world file and to yaml/bitmaps folder for map_server. Open the yaml file that is created by the map_saver, put origin to zero so that the origin will be the lower left corner and put it in the yaml folder. Do not forget to change the path to the mapfile.pgm (in this example bitmaps folder is added).

```
image: bitmaps/mapfile.pgm
resolution: 0.068000
origin: [0, 0, 0.000000]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.196
```

The Stage's world file should have defined size of the floorplan and pose of the floorplan which defines the coordinate frame. There is a parameter for a map resolution but it is ignored. Put the right size of the floorplan by calculating resolution*numpixels_col x resolution*numpixels_row and put the origin to half of it. 

Here is the example for the IML lab floorplan. The size of the map in pixels is 7057 x 2308 and the size of the pixel (resolution) is 0.007 m (the resolution is written in IMLlab.yaml file) so the Stage world file looks as follows:

```
floorplan( 
bitmap "elements/smartface_topologie_entwurf.png"
#map_resolution     0.068 
  pose [24.6995 8.078 0.000 0.000] 
  size [49.399 16.156 0.800] 
  name "IMLlab"
)
```
Maybe you need to set the threshold of the image to have all the obstacle visible also in rviz, as was the case with this png file, since Stage has different thresholds for occupied pixels.


# <a name="mapupdates">Map updates</a>

The map updates are the downsampled laser points to the fine resolution grid, e.g. 0.1 m cell size, which are not mapped in the initial map.
It is supposed that there is no moving obstacles in the environment, only the static ones that are not mapped in the initial map.
So if you move the green box in the Stage simulator the trail of the obstacle will be mapped and also remembered in the list of new obstacles.
By now, the list of new obstacles is never reset. This should be changed when taking into account dynamic obstacles.
The message that is returned is the simple array of coordinates (x,y) of all unmapped obstacles which are snapped to the fine resolution grid points:

NewObstacles.msg

	Header header # standard ROS header
	float64[] x		# x coordinate of the cell centre
	float64[] y		# y coordinate of the cell centre
	
The package _mapupdates_ creates the fine gridmap for keeping track of mapped and unmapped obstacles that needs to be run at the AGV computer. It is connected to the laser sensor on the AGV computer and it is important to set the right topic for the laser.
It is also important to set the robot id as args of the package, as is the example here:
```
<launch>

    <node name="mapup" pkg="mapupdates" type="mapup" output="screen" args="0" >
        <param name="cell_size" type="double" value="0.1" />
        <param name="scan_topic" value="/base_scan" />
    </node>

</launch>
```
The topic that is returned contains the robot ID, e.g. if set to 0 the topic will be /robot_0/newObstacles and the example of the echo of it is:
```
$rostopic echo /robot_0/newObstacles
header: 
  seq: 438
  stamp: 
    secs: 53
    nsecs: 600000000
  frame_id: "map"
x: [1.55, 0.6500000000000001, 0.6500000000000001, 0.6500000000000001, 0.6500000000000001, 0.7500000000000001, 0.7500000000000001, 0.6500000000000001, 1.1500000000000001, 1.1500000000000001, 1.35, 1.35, 1.9500000000000002, 2.05, 2.05, 0.25, 0.55, 0.25, 0.25, 0.25, 0.25, 0.25, 0.35000000000000003, 0.7500000000000001, 0.6500000000000001, 0.7500000000000001, 0.7500000000000001, 0.7500000000000001, 0.7500000000000001, 0.7500000000000001, 1.1500000000000001, 1.7500000000000002, 2.05, 2.05, 2.05, 2.05, 2.05, 2.05, 2.25, 2.45, 2.55, 3.05, 3.05, 2.85, 2.75, 2.85, 2.85, 2.85, 3.05, 3.25, 3.25, 3.35, 3.45, 3.55, 3.65, 3.95, 3.95, 4.05, 4.15, 4.25, 4.55, 4.8500000000000005, 4.95, 4.95, 5.05, 5.15, 5.15, 5.25, 5.45, 6.3500000000000005, 6.45, 6.65, 6.75, 6.75, 6.95, 7.15, 7.25, 7.95, 8.15, 8.250000000000002, 8.250000000000002, 8.250000000000002, 8.450000000000001, 8.450000000000001, 8.250000000000002, 8.350000000000001, 8.850000000000001, 8.450000000000001, 8.450000000000001, 9.250000000000002, 9.350000000000001, 9.450000000000001, 9.55, 8.55, 8.65, 9.15, 9.250000000000002, 9.650000000000002, 9.750000000000002, 9.750000000000002, 9.750000000000002, 8.55, 8.55, 8.55, 8.55, 8.65, 8.55, 8.250000000000002, 8.05, 8.05, 8.15, 8.15, 8.250000000000002, 9.750000000000002, 9.750000000000002, 10.150000000000002, 9.950000000000001, 12.850000000000001, 12.750000000000002, 12.650000000000002, 10.05, 10.05, 10.05, 9.850000000000001, 8.65, 8.55, 8.450000000000001, 8.450000000000001, 8.350000000000001, 8.250000000000002, 0.0, 0.0, 8.05, 7.3500000000000005, 7.15, 7.15, 7.15, 7.05, 6.95, 6.8500000000000005, 6.45, 6.25, 5.75, 5.65, 5.55, 5.45, 5.3500000000000005, 5.25, 4.95, 4.55, 2.05, 1.9500000000000002, 2.35, 2.35, 2.75, 3.05, 3.05, 2.85, 2.85, 2.85, 2.75, 1.55, 1.55, 1.55, 1.4500000000000002, 0.25, 0.25, 1.05, 1.2500000000000002, 1.35, 1.55, 1.55]
y: [4.25, 4.05, 3.95, 3.85, 3.75, 3.75, 3.65, 3.55, 3.65, 3.55, 3.55, 3.45, 3.55, 3.55, 3.45, 2.95, 2.85, 2.65, 2.55, 2.45, 2.35, 2.25, 2.25, 2.25, 2.15, 2.15, 2.05, 1.9500000000000002, 1.85, 1.7500000000000002, 1.7500000000000002, 1.7500000000000002, 1.7500000000000002, 1.6500000000000001, 1.55, 1.4500000000000002, 1.35, 1.2500000000000002, 1.35, 1.35, 1.2500000000000002, 1.35, 1.2500000000000002, 0.9500000000000001, 0.6500000000000001, 0.6500000000000001, 0.55, 0.45, 0.45, 0.45, 0.35000000000000003, 0.35000000000000003, 0.35000000000000003, 0.35000000000000003, 0.35000000000000003, 0.25, 0.15000000000000002, 0.15000000000000002, 0.25, 0.15000000000000002, 0.35000000000000003, 1.05, 1.05, 0.9500000000000001, 1.05, 0.8500000000000001, 0.7500000000000001, 0.45, 0.35000000000000003, 0.45, 0.45, 0.45, 0.35000000000000003, 1.2500000000000002, 1.2500000000000002, 1.1500000000000001, 1.05, 0.45, 0.45, 0.35000000000000003, 0.45, 0.55, 0.35000000000000003, 0.55, 1.05, 1.05, 0.45, 1.05, 1.1500000000000001, 0.25, 0.25, 0.25, 0.25, 1.6500000000000001, 1.7500000000000002, 1.85, 1.85, 1.85, 1.7500000000000002, 1.85, 1.9500000000000002, 3.25, 3.35, 3.45, 3.55, 3.55, 3.85, 3.95, 4.55, 4.65, 4.65, 4.75, 4.8500000000000005, 5.05, 5.15, 5.55, 5.55, 6.8500000000000005, 6.95, 6.95, 6.8500000000000005, 7.15, 7.3500000000000005, 7.3500000000000005, 7.3500000000000005, 7.3500000000000005, 7.3500000000000005, 7.45, 7.45, 7.45, 0.0, 0.0, 7.45, 7.15, 7.15, 7.25, 7.3500000000000005, 7.3500000000000005, 7.3500000000000005, 7.3500000000000005, 5.65, 5.55, 5.55, 5.55, 5.55, 5.55, 5.65, 5.65, 5.65, 5.65, 6.95, 6.95, 6.55, 6.45, 6.25, 5.95, 5.8500000000000005, 5.8500000000000005, 5.75, 5.65, 5.55, 5.8500000000000005, 5.75, 5.65, 5.55, 5.75, 5.65, 5.25, 4.95, 4.75, 4.65, 4.55]
``` 

First start the AMCL localization in the known map and the simulator Stage in which laser data are simulated.
Then start the package mapupdates where new laser readings are compared to the cells of the gridmap. The package mapupdates converts the local laser readings to a global coordinate frame which can be visualized in rviz. This is used to test if the transformation is done correctly and marker points from topic /globalpoints_marker should be aligned in rviz over the simulated laser scan data.
And finaly, start maptogridmap to visualize the new obstacles and topology updates. The package maptogridmap is subscribed to topic /robot_0/newObstacles and checks if points belong to the free grid cell and changes its occupancy accordingly. Nodes and edges are updated too. 

```
terminal 1: roslaunch lam_simulator AndaOmnidriveamcltestZagrebdemo.launch
terminal 2: roslaunch mapupdates startmapupdates.launch
terminal 3: roslaunch maptogridmap startmaptogridmap.launch
```
To read the topic in your own package you need to subscribe to it, include the header of the message, and write a message callback (as is the case in the maptogridmap package).

* subscribe to a topic /robot_0/newObstacles
```
  ros::Subscriber gmu_sub = nh.subscribe("/robot_0/newObstacles",1,newObstaclesCallback);

```
* include the header of the message in your header file or in the cpp where you are writting the message callback:
```
#include <mapupdates/NewObstacles.h>
```
* write a message callback (in this example we just rewrite the global variable obstacles that is used in the main program):
```
mapupdates::NewObstacles obstacles;	#global variable
void newObstaclesCallback(const mapupdates::NewObstaclesConstPtr& msg)
{
	obstacles.x.clear();
	obstacles.y.clear();
	for (int i =0; i<msg->x.size(); i++){
		obstacles.x.push_back(msg->x[i]);
		obstacles.y.push_back(msg->y[i]);
	}
}
```
* maybe you also need to add in CMakeLists.txt of your package the line _mapupdates_ in find_package function otherwise the compiler will complain ti can not find the header file:
```
find_package(catkin REQUIRED COMPONENTS
  std_msgs
  nav_msgs
  geometry_msgs
  message_generation
  roscpp
  tf
  maptogridmap
  mapupdates
)
```
Another example of subscribing to a topic /robot_0/newObstacles is in _maplistener_ package:
```
newobs_sub = nh_.subscribe("/robot_0/newObstacles",1,&VisualizationPublisherGML::newObstaclesCallback, this);
```
* write a message callback for visualizing the marker topic /newobstacles_markerListener in rviz:
```
void VisualizationPublisherGML::newObstaclesCallback(const mapupdates::NewObstaclesConstPtr& msg)
{
  glp.points.clear();
  geometry_msgs::Point p; 
	for (int i =0; i<msg->x.size(); i++){
		p.x=msg->x[i];
		p.y=msg->y[i];
		glp.points.push_back(p);
	}
}
```
* start maplistener to see the marker topic in rviz:
```
terminal 4: rosrun maplistener mapls
```

# Examples
## Testing if ROS topics for Nodes and Edges are sent to Orion Context Broker:
On opil server make a clean start of context broker:
```
sudo docker-compose down
sudo docker-compose up
```
Check in firefox if http://OPIL_SERVER_IP:1026/v2/entities is blank (replace OPIL_SERVER_IP with the correct IP address).

On machine 1 start:

* Uncomment the map you want to use (MP, IML or Anda) and comment the rest of maps in startmapserver.launch.
```
terminal 1: roslaunch maptogridmap startmapserver.launch
terminal 2: roslaunch maptogridmap startmaptogridmap.launch
```
If you want to send the topics through firos, you need to put in firos/config all json files from test/config_files/machine_1:
```
terminal 3: rosrun firos core.py
```

Refresh firefox on http://OPIL_SERVER_IP:1026/v2/entities. There should be under id "map" topics "nodes" and "edges" with their values.


## Testing sending pose with covariance on machine_1
Just start after previous three terminals. If you want to test only sending pose with covariance, repeat the commands in section [Pose with covariance](#poswithcov).
Start the simulation Stage with the amcl localization depending on the map you are using in startmapserver.launch. For example, if you are using MP map use amcl_test_muraplast.launch; if you are using IML map use IMLamcltest.launch; otherwise, if you are using ICENT map use the following:
```
terminal 4: roslaunch lam_simulator AndaOmnidriveamcltestZagrebdemo.launch
```
This starts stage simulator and amcl localization. 
```
terminal 5: roslaunch sensing_and_perception send_posewithcovariance.launch 
```
Now you can refresh firefox on http://OPIL_SERVER_IP:1026/v2/entities.
There should be under id "robot_0" with topic "pose_channel".
Simply move the robot in stage by dragging it with the mouse and refresh the firefox to see the update of pose_channel.


## Testing if topics for Nodes, Edges, and PoseWithCovariance are received on machine_2 through firos
If you want to receive the topics through firos, you need to put in firos/config all json files from test/config_files/machine_2:

```
terminal 1: roscore
terminal 2: rosrun firos core.py
```
_maptogridmap_ package needs to be on the machine_2 - it is not important that the source code is in there but only that msg files and CMakeLists.txt compiling them are there.
Now you are able to echo all ros topics:
```
rostopic echo /map/nodes
rostopic echo /map/edges
rostopic echo /robot_0/pose_channel
```
* <a name="exampleannot">Example output for the ICENT map - Nodes with loaded annotations</a>
<!--* Example output for the ICENT map - Nodes-->

```
$rostopic echo /map/nodes
header: 
  seq: 104
  stamp: 
    secs: 65
    nsecs: 300000000
  frame_id: "map"
info: 
  map_load_time: 
    secs: 0
    nsecs:         0
  resolution: 1.0
  width: 14
  height: 8
  origin: 
    position: 
      x: 0.0
      y: 0.0
      z: 0.0
    orientation: 
      x: 0.0
      y: 0.0
      z: 0.0
      w: 0.0
x: [1.5, 2.5, 2.5, 2.9, 3.5, 3.75, 3.5, 3.5, 3.5, 3.5, 4.5, 4.5, 4.5, 4.5, 4.5, 5.5, 5.5, 5.5, 5.5, 6.4, 6.5, 6.9, 7.5, 7.5, 7.5, 7.5, 7.5, 8.5, 8.5, 9.5, 10.5, 11.5, 11.5, 11.5, 11.5, 11.5, 11.5, 11.5, 12.5, 12.5, 12.5, 12.5, 12.5, 13.5, 13.5, 13.5, 13.5, 13.5, 13.5, 13.5]
y: [2.5, 2.5, 3.5, 4.6, 1.5, 2.51, 3.5, 4.5, 5.5, 6.5, 1.5, 2.5, 3.5, 4.5, 6.5, 1.5, 2.5, 3.5, 4.5, 2.51, 3.5, 4.3, 2.5, 3.5, 4.5, 5.5, 6.5, 5.5, 6.5, 6.5, 1.5, 0.5, 1.5, 2.5, 3.5, 4.5, 5.5, 6.5, 0.5, 1.5, 2.5, 3.5, 4.5, 0.5, 1.5, 2.5, 3.5, 4.5, 5.5, 7.5]
theta: [0.0, 0.0, 0.0, 30.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 90.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
name: [vertex_10, vertex_18, vertex_19, P1, vertex_25, loadingArea, vertex_27, vertex_28,
  vertex_29, vertex_30, vertex_33, vertex_34, vertex_35, vertex_36, vertex_38, vertex_41,
  vertex_42, vertex_43, vertex_44, unloadingArea, vertex_51, waitingArea, vertex_58,
  vertex_59, vertex_60, vertex_61, vertex_62, vertex_69, vertex_70, vertex_78, vertex_81,
  vertex_88, vertex_89, vertex_90, vertex_91, vertex_92, vertex_93, vertex_94, vertex_96,
  vertex_97, vertex_98, vertex_99, vertex_100, vertex_104, vertex_105, vertex_106,
  vertex_107, vertex_108, vertex_109, vertex_111]
uuid: [57c319ef-5830-4035-bfd4-bb4852bf7f3e, c83f8271-c67c-48df-b654-b69930846891, 15818a4b-6749-4d8f-a550-15a6291b9d28,
  70536798-2680-43f8-a415-64d70dd249dd, 4931ab8c-9f2f-4913-8768-98edbd75f543, 6cf2954e-dafa-45f1-ba21-3a4f64909153,
  1173257d-2ece-4dd1-98ba-9d762201b592, c33bc043-55da-4689-aec4-d82ec4a31197, 89f004df-c2fd-43cf-bb61-6d7bf83d50ea,
  757ac47e-33b3-4acf-ad85-81c431667806, 0ca96b41-e949-426b-afab-c183e7be1ff2, 9b369197-9275-4db4-8b8b-5ebabaa7b833,
  3e29ca55-bb7d-4b91-a9c9-4f5cbfff29cb, 275b3195-152f-4659-8b96-9e4683a1af91, 172086aa-5067-4f74-9a65-318872d9f9e6,
  97fddd32-361e-4c8c-b1a6-c99eab4da583, 9b2a0e09-14df-4667-858a-4395dcc85170, 123f665a-5cab-41bf-a872-bf998f647f5c,
  d27810f6-689a-47b6-a5c9-d5462baa9dd9, 2c2be0b8-a750-499f-89bb-83c178a0809b, d4c709ab-fa4a-4b87-9f43-269af26f436a,
  6416975f-747f-439e-9bea-987c7e5ba9cc, 4bd3cc9d-c3c1-468a-934f-430a2863fbc7, 80c3750b-536a-464f-bd79-f2356ef68f58,
  a3d23430-38ed-42c4-8d9e-2d638a1697bd, 01fd7093-4716-45d2-8e8b-6ddeb04f5caf, 715e4b36-134b-4c0d-83ee-398585d2c296,
  d98e4e0f-7be1-40c4-b355-01d8edee2919, 0a660726-83cb-4b09-99f3-57df923b46a2, 2f543486-e47d-48e7-89bd-08743d509c55,
  06619538-37af-47ec-a642-657642f0eca4, 192b0899-c1d4-4458-a9b8-c4da5fc51ba6, dbbc46e7-93f7-492a-b758-d92a398f4e65,
  3d187492-0862-44dd-b28e-2d6e53b7ede4, a5aa74ab-9949-4a69-951b-1192eb9f04f3, a0178b30-50ef-4d69-9b32-f4e6a4bd8a11,
  51190db8-d13b-4838-8a09-452d9f46c030, e62dd631-99aa-40eb-8001-11b720823bb1, eff5f5a0-54db-4f38-a35e-75ef77abe4b9,
  3f1d9f14-9c96-421e-ab47-169007dd4fae, d8c89d2c-9a69-4dbf-a113-9c9d0d5a9bd4, d3a54686-f687-4791-845f-71111f390770,
  36a0a772-fe64-46aa-8f12-252dba151445, d0b5b5fb-5242-4bbc-b893-ced7727b334c, 2883a9ef-d003-4446-b634-929e055b8465,
  0625388d-b9c9-4c28-8b73-bd3637f686f4, fd27eca8-6744-4a75-be29-e9a0adf189c0, 28023451-3b2c-4b4e-9d07-171d3eb2acc4,
  ff9fa735-260f-4ec3-b2a4-9896ef14fc27, e014d0b0-1130-4e02-b402-1aab060a9a10]
```

* Example output for the ICENT map - Edges

```
$rostopic echo /map/edges
header: 
  seq: 4893
  stamp: 
    secs: 544
    nsecs: 200000000
  frame_id: "map"
uuid_src: [57c319ef-5830-4035-bfd4-bb4852bf7f3e, c83f8271-c67c-48df-b654-b69930846891, c83f8271-c67c-48df-b654-b69930846891,
  15818a4b-6749-4d8f-a550-15a6291b9d28, 15818a4b-6749-4d8f-a550-15a6291b9d28, 70536798-2680-43f8-a415-64d70dd249dd,
  4931ab8c-9f2f-4913-8768-98edbd75f543, 4931ab8c-9f2f-4913-8768-98edbd75f543, 6cf2954e-dafa-45f1-ba21-3a4f64909153,
  6cf2954e-dafa-45f1-ba21-3a4f64909153, 1173257d-2ece-4dd1-98ba-9d762201b592, 1173257d-2ece-4dd1-98ba-9d762201b592,
  c33bc043-55da-4689-aec4-d82ec4a31197, c33bc043-55da-4689-aec4-d82ec4a31197, 89f004df-c2fd-43cf-bb61-6d7bf83d50ea,
  757ac47e-33b3-4acf-ad85-81c431667806, 0ca96b41-e949-426b-afab-c183e7be1ff2, 0ca96b41-e949-426b-afab-c183e7be1ff2,
  9b369197-9275-4db4-8b8b-5ebabaa7b833, 9b369197-9275-4db4-8b8b-5ebabaa7b833, 3e29ca55-bb7d-4b91-a9c9-4f5cbfff29cb,
  3e29ca55-bb7d-4b91-a9c9-4f5cbfff29cb, 275b3195-152f-4659-8b96-9e4683a1af91, 97fddd32-361e-4c8c-b1a6-c99eab4da583,
  9b2a0e09-14df-4667-858a-4395dcc85170, 9b2a0e09-14df-4667-858a-4395dcc85170, 123f665a-5cab-41bf-a872-bf998f647f5c,
  123f665a-5cab-41bf-a872-bf998f647f5c, d27810f6-689a-47b6-a5c9-d5462baa9dd9, 2c2be0b8-a750-499f-89bb-83c178a0809b,
  2c2be0b8-a750-499f-89bb-83c178a0809b, d4c709ab-fa4a-4b87-9f43-269af26f436a, d4c709ab-fa4a-4b87-9f43-269af26f436a,
  6416975f-747f-439e-9bea-987c7e5ba9cc, 4bd3cc9d-c3c1-468a-934f-430a2863fbc7, 80c3750b-536a-464f-bd79-f2356ef68f58,
  a3d23430-38ed-42c4-8d9e-2d638a1697bd, 01fd7093-4716-45d2-8e8b-6ddeb04f5caf, 01fd7093-4716-45d2-8e8b-6ddeb04f5caf,
  715e4b36-134b-4c0d-83ee-398585d2c296, d98e4e0f-7be1-40c4-b355-01d8edee2919, 0a660726-83cb-4b09-99f3-57df923b46a2,
  06619538-37af-47ec-a642-657642f0eca4, 192b0899-c1d4-4458-a9b8-c4da5fc51ba6, dbbc46e7-93f7-492a-b758-d92a398f4e65,
  dbbc46e7-93f7-492a-b758-d92a398f4e65, 3d187492-0862-44dd-b28e-2d6e53b7ede4, 3d187492-0862-44dd-b28e-2d6e53b7ede4,
  a5aa74ab-9949-4a69-951b-1192eb9f04f3, a5aa74ab-9949-4a69-951b-1192eb9f04f3, a0178b30-50ef-4d69-9b32-f4e6a4bd8a11,
  a0178b30-50ef-4d69-9b32-f4e6a4bd8a11, 51190db8-d13b-4838-8a09-452d9f46c030, eff5f5a0-54db-4f38-a35e-75ef77abe4b9,
  3f1d9f14-9c96-421e-ab47-169007dd4fae, 3f1d9f14-9c96-421e-ab47-169007dd4fae, d8c89d2c-9a69-4dbf-a113-9c9d0d5a9bd4,
  d8c89d2c-9a69-4dbf-a113-9c9d0d5a9bd4, d3a54686-f687-4791-845f-71111f390770, d3a54686-f687-4791-845f-71111f390770,
  36a0a772-fe64-46aa-8f12-252dba151445, d0b5b5fb-5242-4bbc-b893-ced7727b334c, 2883a9ef-d003-4446-b634-929e055b8465,
  0625388d-b9c9-4c28-8b73-bd3637f686f4, fd27eca8-6744-4a75-be29-e9a0adf189c0, 28023451-3b2c-4b4e-9d07-171d3eb2acc4]
uuid_dest: [c83f8271-c67c-48df-b654-b69930846891, 6cf2954e-dafa-45f1-ba21-3a4f64909153, 15818a4b-6749-4d8f-a550-15a6291b9d28,
  1173257d-2ece-4dd1-98ba-9d762201b592, 70536798-2680-43f8-a415-64d70dd249dd, c33bc043-55da-4689-aec4-d82ec4a31197,
  0ca96b41-e949-426b-afab-c183e7be1ff2, 6cf2954e-dafa-45f1-ba21-3a4f64909153, 9b369197-9275-4db4-8b8b-5ebabaa7b833,
  1173257d-2ece-4dd1-98ba-9d762201b592, 3e29ca55-bb7d-4b91-a9c9-4f5cbfff29cb, c33bc043-55da-4689-aec4-d82ec4a31197,
  275b3195-152f-4659-8b96-9e4683a1af91, 89f004df-c2fd-43cf-bb61-6d7bf83d50ea, 757ac47e-33b3-4acf-ad85-81c431667806,
  172086aa-5067-4f74-9a65-318872d9f9e6, 97fddd32-361e-4c8c-b1a6-c99eab4da583, 9b369197-9275-4db4-8b8b-5ebabaa7b833,
  9b2a0e09-14df-4667-858a-4395dcc85170, 3e29ca55-bb7d-4b91-a9c9-4f5cbfff29cb, 123f665a-5cab-41bf-a872-bf998f647f5c,
  275b3195-152f-4659-8b96-9e4683a1af91, d27810f6-689a-47b6-a5c9-d5462baa9dd9, 9b2a0e09-14df-4667-858a-4395dcc85170,
  2c2be0b8-a750-499f-89bb-83c178a0809b, 123f665a-5cab-41bf-a872-bf998f647f5c, d4c709ab-fa4a-4b87-9f43-269af26f436a,
  d27810f6-689a-47b6-a5c9-d5462baa9dd9, 6416975f-747f-439e-9bea-987c7e5ba9cc, 4bd3cc9d-c3c1-468a-934f-430a2863fbc7,
  d4c709ab-fa4a-4b87-9f43-269af26f436a, 80c3750b-536a-464f-bd79-f2356ef68f58, 6416975f-747f-439e-9bea-987c7e5ba9cc,
  a3d23430-38ed-42c4-8d9e-2d638a1697bd, 80c3750b-536a-464f-bd79-f2356ef68f58, a3d23430-38ed-42c4-8d9e-2d638a1697bd,
  01fd7093-4716-45d2-8e8b-6ddeb04f5caf, d98e4e0f-7be1-40c4-b355-01d8edee2919, 715e4b36-134b-4c0d-83ee-398585d2c296,
  0a660726-83cb-4b09-99f3-57df923b46a2, 0a660726-83cb-4b09-99f3-57df923b46a2, 2f543486-e47d-48e7-89bd-08743d509c55,
  dbbc46e7-93f7-492a-b758-d92a398f4e65, dbbc46e7-93f7-492a-b758-d92a398f4e65, 3f1d9f14-9c96-421e-ab47-169007dd4fae,
  3d187492-0862-44dd-b28e-2d6e53b7ede4, d8c89d2c-9a69-4dbf-a113-9c9d0d5a9bd4, a5aa74ab-9949-4a69-951b-1192eb9f04f3,
  d3a54686-f687-4791-845f-71111f390770, a0178b30-50ef-4d69-9b32-f4e6a4bd8a11, 36a0a772-fe64-46aa-8f12-252dba151445,
  51190db8-d13b-4838-8a09-452d9f46c030, e62dd631-99aa-40eb-8001-11b720823bb1, 3f1d9f14-9c96-421e-ab47-169007dd4fae,
  2883a9ef-d003-4446-b634-929e055b8465, d8c89d2c-9a69-4dbf-a113-9c9d0d5a9bd4, 0625388d-b9c9-4c28-8b73-bd3637f686f4,
  d3a54686-f687-4791-845f-71111f390770, fd27eca8-6744-4a75-be29-e9a0adf189c0, 36a0a772-fe64-46aa-8f12-252dba151445,
  28023451-3b2c-4b4e-9d07-171d3eb2acc4, 2883a9ef-d003-4446-b634-929e055b8465, 0625388d-b9c9-4c28-8b73-bd3637f686f4,
  fd27eca8-6744-4a75-be29-e9a0adf189c0, 28023451-3b2c-4b4e-9d07-171d3eb2acc4, ff9fa735-260f-4ec3-b2a4-9896ef14fc27]
name: [edge_10_18, edge_18_26, edge_18_19, edge_19_27, edge_19_20, edge_20_28, edge_25_33,
  edge_25_26, edge_26_34, edge_26_27, edge_27_35, edge_27_28, edge_28_36, edge_28_29,
  edge_29_30, edge_30_38, edge_33_41, edge_33_34, edge_34_42, edge_34_35, edge_35_43,
  edge_35_36, edge_36_44, edge_41_42, edge_42_50, edge_42_43, edge_43_51, edge_43_44,
  edge_44_52, edge_50_58, edge_50_51, edge_51_59, edge_51_52, edge_52_60, edge_58_59,
  edge_59_60, edge_60_61, edge_61_69, edge_61_62, edge_62_70, edge_69_70, edge_70_78,
  edge_81_89, edge_88_89, edge_89_97, edge_89_90, edge_90_98, edge_90_91, edge_91_99,
  edge_91_92, edge_92_100, edge_92_93, edge_93_94, edge_96_97, edge_97_105, edge_97_98,
  edge_98_106, edge_98_99, edge_99_107, edge_99_100, edge_100_108, edge_104_105, edge_105_106,
  edge_106_107, edge_107_108, edge_108_109]
uuid: [54d7fef2-4b84-4af3-817e-1372eb7ce426, c85274d8-e8a4-4381-998d-5d2b29cdac07, 7fd651fa-5658-4dfb-b077-e43cac0e0947,
  dd35b234-3d93-473e-a3e1-e36c2a548ab0, f2b0062c-2e84-4e3b-a012-4a3d34c6fcec, 25d00572-22a1-4b78-b708-6ad77e1ad571,
  2faeb79f-f610-4094-abe5-ec3c497adac7, dab00789-1795-4e4a-a501-5e2f6a4a52dd, d59148f3-045f-4e46-b1bf-9a2b562e14f2,
  c5fc202f-ad70-4ee0-97f4-2497ec3a0e31, f438e348-3107-4d89-ad80-c3191537ac85, a970bafc-0136-4f9c-ba88-4a2ae7217be8,
  6633880a-a907-4713-ab97-8c26d3ee6595, 1339b945-2149-421f-b2f6-14f7babc4693, a4bf69f8-e2c8-447d-8f7e-56772e69ae5f,
  b4fd8305-45ba-4590-a736-8197bbe5cff3, 166f6e14-489d-469b-96fd-b52c67698750, fdf95d41-7573-4c01-8c02-67dec3e70932,
  b993d323-6b0e-4a21-b2bf-3a2a234386cc, 6ad0a7aa-4a9f-4c62-9a7b-d54fd060a938, adaf3199-30fe-42cb-9972-db8e05385499,
  0c1666af-acd9-4641-9280-580be1765365, b76bef74-4c10-4ba7-9261-1d856301656d, de491bf0-6ea8-4347-a109-30b87a6dddad,
  8694d196-d938-4fc8-812d-9c58a58c96bc, f01155be-5153-4a70-939b-98186d7a9e7b, 9e6162da-b6ba-4b51-af75-5e71577dac7d,
  bddf9946-26ac-4dde-b8b5-f5f8458c8a38, 73d10460-c5ad-4bb0-bc14-c3a544b01994, b63fce9a-2053-4f09-9d9d-4aee8cfab15d,
  9ff8cc1f-d497-402f-9164-8abed94506d0, c5815d3c-821f-4c35-9e2a-c824fe9d94d9, 103ec47e-04ac-44fc-bc60-58b7898cfd5f,
  4ea77d01-1585-4982-bf00-971dec77451f, a0df856c-50bc-4a9a-abc5-149bc2f733d3, 95d8888d-0994-46ef-9205-a02c0ad0dd41,
  8520780f-5258-42f1-9cca-5d0448ea58c3, f2276c4e-e22e-4352-a76d-9f244e9f6e9e, 09f8210b-c372-4d4c-83a3-f86b84bcd13b,
  9c1cdbb9-7d7f-43e4-a751-a1410151c840, b8c71526-63ea-49bb-bd1b-d5af1801bcf4, 3d3711cd-ef0f-4543-b852-2cfb3f178bbd,
  158fd367-7d20-42d2-bfdd-6c118d30aebf, 1388dd39-086f-40d6-9a5b-e824397bef70, bfc57036-421e-48c7-80e2-90e967d6f355,
  7da849ab-4965-465d-af76-18e141b81256, 2a4aeb78-0186-4416-9b87-31369d5396f2, abee7eec-8a3c-4f97-b239-6d34c954e488,
  fc8b61aa-50c1-435a-86a2-27b9eb74e165, 83ca153c-dde5-4752-a868-57f5f599d758, f1fad6f3-8b69-4ee5-861b-64a9205a10d6,
  3b8685e6-e917-4c50-9358-a2b196c2ebd3, 0536b4d1-e337-44ac-bfaf-e4100ae90a23, 792a5bef-64ea-46ff-8386-69e93db3aa33,
  e583e7cf-084a-41b2-8810-dd461c695ffe, e1e966ea-050f-4c36-8a1b-c187a50b8d25, ee3fb5bc-a2b5-4c8e-9cbf-243fb8ebc023,
  45a6b2f7-2911-4576-b80d-931a705b2718, 2b03d23d-69b5-4a68-b312-fbd401c21e7b, f35f74fd-d6ae-479d-8833-8a502047ec93,
  6fbd5fa7-c1bd-493d-9a41-02a5e377df79, 5369a3da-b5f5-4726-b095-2e77c372eb4b, 6919e25a-b105-4517-90bf-9e92549cb706,
  4fbf645f-330e-4cc3-8d8e-d5330269b4e5, 12855305-90fd-46be-b5cf-93d669e5c9e8, 8c0d54c1-f40b-4612-82c9-5190bbf54d3a]
```

* <a name="examplepose">Example output for the ICENT map - Pose with covariance</a>
<!--* Example output for the ICENT map - Pose with covariance-->
```
$rostopic echo /robot_0/pose_channel
header: 
  seq: 221
  stamp: 
    secs: 667
    nsecs: 200000000
  frame_id: "map"
pose: 
  pose: 
    position: 
      x: 5.89260851198
      y: 4.54273166596
      z: 0.0
    orientation: 
      x: 0.0
      y: 0.0
      z: 0.000341716300675
      w: 0.999999941615
  covariance: [0.2280276789742004, 0.00121444362006784, 0.0, 0.0, 0.0, 0.0, 0.00121444362006784, 0.2095520253272909, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.06568863093933444]
```

* Example output for the IML map - Nodes (todo replace with new topic echo, theta is introduced)

```
$rostopic echo /map/nodes
header: 
  seq: 196
  stamp: 
    secs: 1548412279
    nsecs: 782414849
  frame_id: "map"
info: 
  map_load_time: 
    secs: 1548412279
    nsecs: 782414480
  resolution: 2.0
  width: 25
  height: 9
  origin: 
    position: 
      x: 0.0
      y: 0.0
      z: 0.0
    orientation: 
      x: 0.0
      y: 0.0
      z: 0.0
      w: 0.0
x: [1.0, 3.0, 3.0, 3.0, 3.0, 3.0, 3.0, 5.0, 5.0, 5.0, 5.0, 5.0, 5.0, 5.0, 7.0, 7.0, 7.0, 7.0, 7.0, 7.0, 9.0, 9.0, 9.0, 9.0, 9.0, 9.0, 11.0, 11.0, 11.0, 11.0, 11.0, 11.0, 13.0, 13.0, 13.0, 13.0, 13.0, 13.0, 15.0, 15.0, 15.0, 15.0, 15.0, 15.0, 17.0, 17.0, 17.0, 17.0, 19.0, 19.0, 19.0, 19.0, 21.0, 21.0, 21.0, 21.0, 21.0, 21.0, 23.0, 23.0, 23.0, 23.0, 23.0, 23.0, 25.0, 25.0, 25.0, 25.0, 27.0, 27.0, 27.0, 27.0, 29.0, 29.0, 29.0, 29.0, 29.0, 29.0, 31.0, 31.0, 31.0, 31.0, 33.0, 33.0, 35.0, 35.0, 35.0, 35.0, 37.0, 37.0, 37.0, 37.0, 39.0, 39.0, 39.0, 39.0, 39.0, 39.0, 41.0, 41.0, 41.0, 41.0, 43.0, 43.0, 43.0, 43.0, 45.0, 45.0, 45.0, 45.0, 47.0, 47.0, 47.0, 47.0, 49.0]
y: [17.0, 3.0, 5.0, 7.0, 9.0, 11.0, 13.0, 3.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 9.0, 11.0, 13.0, 17.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 9.0, 11.0, 13.0, 17.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 17.0, 9.0, 17.0, 9.0, 11.0, 13.0, 17.0, 5.0, 7.0, 9.0, 11.0, 3.0, 5.0, 7.0, 9.0, 11.0, 17.0, 3.0, 5.0, 7.0, 17.0, 3.0, 5.0, 7.0, 17.0, 3.0, 5.0, 7.0, 17.0, 3.0, 5.0, 7.0, 17.0, 17.0]
name: [vertex_8, vertex_10, vertex_11, vertex_12, vertex_13, vertex_14, vertex_15, vertex_19,
  vertex_20, vertex_21, vertex_22, vertex_23, vertex_24, vertex_26, vertex_29, vertex_30,
  vertex_31, vertex_32, vertex_33, vertex_35, vertex_38, vertex_39, vertex_40, vertex_41,
  vertex_42, vertex_44, vertex_47, vertex_48, vertex_49, vertex_50, vertex_51, vertex_53,
  vertex_56, vertex_57, vertex_58, vertex_59, vertex_60, vertex_62, vertex_65, vertex_66,
  vertex_67, vertex_68, vertex_69, vertex_71, vertex_76, vertex_77, vertex_78, vertex_80,
  vertex_85, vertex_86, vertex_87, vertex_89, vertex_92, vertex_93, vertex_94, vertex_95,
  vertex_96, vertex_98, vertex_101, vertex_102, vertex_103, vertex_104, vertex_105,
  vertex_107, vertex_112, vertex_113, vertex_114, vertex_116, vertex_121, vertex_122,
  vertex_123, vertex_125, vertex_128, vertex_129, vertex_130, vertex_131, vertex_132,
  vertex_134, vertex_137, vertex_138, vertex_139, vertex_143, vertex_148, vertex_152,
  vertex_157, vertex_158, vertex_159, vertex_161, vertex_164, vertex_165, vertex_166,
  vertex_167, vertex_172, vertex_173, vertex_174, vertex_175, vertex_176, vertex_179,
  vertex_181, vertex_182, vertex_183, vertex_188, vertex_190, vertex_191, vertex_192,
  vertex_197, vertex_199, vertex_200, vertex_201, vertex_206, vertex_208, vertex_209,
  vertex_210, vertex_215, vertex_224]
uuid: [931954c4-3d54-41fb-8a50-825336423614, 2c0b4f91-8780-45ba-93d5-dd60ab3ea61d, 706b0231-edbb-443e-b80b-330691cdba41,
  23f0fa05-6629-4f0c-a27f-9505006c5fe2, 98aad7a8-2820-4b16-b15d-603f83e1a5da, a0391f1b-8439-4dcd-b06b-3cbb3f2d7dfd,
  a378fbe4-504f-4132-9dd8-68507765de36, b0605c16-a416-4afc-b3c2-821e4fd9d7e5, 0d9f364e-901b-4845-8d6f-76c1122b97d1,
  4dadd7bd-9471-4c57-9b42-9fa89974de16, b51db75b-b22c-466f-8475-26c2ea99b2dc, 24b267fc-ab92-4a71-93c9-9a84785926cb,
  b11c9ae9-c10b-4aef-a299-121d58515840, dfd95cd4-3009-4d13-8587-cb2658db9888, 604067cb-d2f5-47b5-bfa5-8c1e6021b9c8,
  f4712b59-dd34-47ab-bc20-4965f9b36e2a, 8882f309-f497-40df-8cae-b358f53f1524, ce504224-af83-4d89-be6d-f59d6f114597,
  6028db74-70af-40de-b62b-fc0c113f07ba, f1b3ba92-3bba-47ea-9e8f-a248057bc296, c48e49be-1f97-4218-9870-3b73490945b0,
  ade7bff4-1c5f-4009-8f3c-9d35c57a9936, f2c90bdc-8325-446a-ae8a-94f1f0bce245, f63c03ca-f261-4c80-be3a-64f914e1e714,
  c10a9f4a-e6ae-4ae7-8eb5-7c2acb09dd89, 173d07bb-2512-4ec0-87e8-b4ba588b2781, 7dde7e83-e1a7-4e8b-9675-ef7577bdcb86,
  2f529d61-3452-4472-9d6a-4904f61aaea2, 39ff8173-1e10-488f-99f4-6bac128918b5, ef72e9b0-f87b-4ba8-af44-1c3ac8613258,
  44892486-e918-44c3-b42c-9771db99aa8c, 5abb38f3-a5ce-48c5-a1d2-609ffb791a69, c41a6308-4f6f-4073-871e-91b874d5c81f,
  49f57e6d-2474-4937-be38-f395aeaf21aa, e48dae03-49a4-4f7d-9ab8-1e6184f6a575, b26ededc-2ac5-47b3-928f-3f93f3ae3a4e,
  8364bb0f-00c3-40ed-855c-9033dc960649, c5bdffb1-2e50-42d0-a5be-3f3a4023e7c4, 42c3a87a-68d9-45df-8e4e-091a469b22cb,
  2f31e597-7c8a-4cdf-a4f6-113686a031df, a44f64b3-27c6-4436-a3a2-b49bb07cd974, 2fe86d4c-83a7-4180-9618-a6524ebd2511,
  91f1a197-f7b8-4653-9eb3-4c1a52ee21fb, 4733f7f7-08bc-4678-86f2-e750edc0a510, 4356e371-d349-479b-a9c1-abb22346a6fc,
  2189297d-90fa-435f-acd1-63f7af2b5f6b, 690f55e0-87f6-4c5d-8a56-b4fe367ca62b, 8bd310ba-9a12-4c5a-8f57-3bef729ed76e,
  c63be3e6-6a3f-4621-a118-0a216b7c0e0b, 44333768-5fa7-4dad-a870-f7be07f0afa0, 8ff3b6db-241b-4c9a-9b3f-d436fc8b79c5,
  eb5f0148-97ba-45b5-a7a9-d7d0cbdfae26, 766478dd-f712-404a-b374-f0f85796719b, 5c9e1983-243e-4624-9c3c-a0f52932f22f,
  f96bef75-78fb-41ba-8f75-f5f8f5db2adf, 91a2b1da-ec32-475e-b9b6-855972638751, 9024fce9-eb5d-48b4-9e70-16a693b8fd32,
  d7776ffe-ed0f-4368-9434-e7492b65d5b7, 6629a3d6-14a1-4ab7-a6a0-d3bd7f7303e1, 48ce49bb-1940-4380-b22a-9579e2cd5169,
  905e775f-8f6b-4303-84ef-f791ccecb448, 5db5eb57-df39-471a-a950-8b27b2b9963c, ced05dba-4c4d-430e-ae1e-4aa1d979843b,
  d012a1f1-5c07-41c8-a446-4a469898c6a3, 66fc4cee-84f2-42e4-aa7f-3986f49f2c60, 64d71e60-591c-4002-9cfa-53502082eaac,
  9716ef04-9d33-4874-adfd-fd6867ab6cfb, ea0728f5-e93c-48c1-976e-54c262d5d8fa, f8019862-df9e-492f-86cf-ab012cd47fa2,
  6f0c1703-4408-4ddd-9bd4-e74d0d57334c, 56cf9ba0-4cf3-4957-b84b-39637fd5e2ae, 9e7ace98-48ac-4da1-bdcb-311851d578a7,
  fb832fba-328c-4024-aa62-116b1cb6c6da, 03ea5b5d-ad04-4e9f-a50b-a722057eb7df, 9dc00c84-fcb1-499b-b734-91ee5a295973,
  e723997f-6aa3-4868-a71b-56aaed817bca, 93be216d-9b42-44b7-8103-5807bd34df42, 271bc16c-85c5-40b7-b4f6-1208d569b08d,
  cbf79ff4-d961-4379-9fc2-0d379b517bd1, e1874ac4-61b4-4238-b1da-6badb3b508ad, 05273595-b5b9-4bd8-8b72-8122d9b79ac8,
  b5ef1bf2-fcea-42d4-bc9e-9f6c6d921ab1, e279f55f-b8d1-4fca-9803-094972f97125, 6405de78-cddd-4fe7-aab1-df11059726e5,
  061eb5fc-c6a5-4575-98b8-15e0710af34d, 4f39ec35-9f6e-421b-94c9-91e2de32015d, 22b4c79a-27e3-4c8e-978c-c8f85c5dd695,
  69894ffd-c7cb-4e92-9423-8c9f66101741, 4c993324-f59f-4c1e-9e32-8e63b3325479, 18a90346-b758-492c-b0c2-921e0e5176e7,
  3fabf6f5-646c-4019-b7b8-043f2b6440f4, 3999dc32-0572-490e-9873-23259cf6a91e, cecce50b-ea2c-44da-bc7b-610cffa56172,
  59fb1959-d979-4821-a376-beedab111400, 42876f7b-5ab7-4809-92a8-95b1de18f99c, 3a464d12-91d7-4ed0-b88e-c556e088a821,
  d117f425-6c6e-478e-b9a0-a2c71754b2da, 7076e76d-fb57-4ab2-b0d7-81fff7e1b300, cb013a0a-92de-4741-8f2f-3b8a5fe87d9f,
  4a7ad181-4ec8-44aa-8e32-dce9784fb177, fdc3777a-0bc3-48ff-ba11-bdde1de03140, 3816520e-86b8-46b0-8507-a40f7ea81e5f,
  68ed02f3-fc91-4e2a-9ea6-33920cd6f6f4, ee309bc1-acf2-4cd7-ba4c-66918dcac234, 2ba32c76-4f47-4952-9932-f1174f305fd3,
  ef157fc4-fad2-461e-90a0-8bd11820a669, e078a718-52af-47b3-b9d8-9fc22fc0fae8, ded0cd7a-61e6-4d23-ac18-3f5f9d5e0a3d,
  5734df8b-09a6-4f7a-b0dd-db0fcf844e8c, 6110350e-34a6-4155-8bfa-ebdb10e18c33, 5759024e-ede9-4619-a43a-3aa722fac84b,
  0ada6574-13be-4b75-97ed-16feae4a911c, 45f31b84-a6a3-47d1-a675-99aa1530827c, 7de556b0-5050-4946-8811-a35eaf890c12,
  d6f4983d-6653-4709-a772-48f81a8bb1b3]
```
* Example output for the IML map - Edges
```
$rostopic echo /map/edges
header: 
  seq: 762
  stamp: 
    secs: 0
    nsecs:         0
  frame_id: "map"
uuid_src: [2c0b4f91-8780-45ba-93d5-dd60ab3ea61d, 2c0b4f91-8780-45ba-93d5-dd60ab3ea61d, 706b0231-edbb-443e-b80b-330691cdba41,
  706b0231-edbb-443e-b80b-330691cdba41, 23f0fa05-6629-4f0c-a27f-9505006c5fe2, 23f0fa05-6629-4f0c-a27f-9505006c5fe2,
  98aad7a8-2820-4b16-b15d-603f83e1a5da, 98aad7a8-2820-4b16-b15d-603f83e1a5da, a0391f1b-8439-4dcd-b06b-3cbb3f2d7dfd,
  a0391f1b-8439-4dcd-b06b-3cbb3f2d7dfd, a378fbe4-504f-4132-9dd8-68507765de36, b0605c16-a416-4afc-b3c2-821e4fd9d7e5,
  0d9f364e-901b-4845-8d6f-76c1122b97d1, 0d9f364e-901b-4845-8d6f-76c1122b97d1, 4dadd7bd-9471-4c57-9b42-9fa89974de16,
  4dadd7bd-9471-4c57-9b42-9fa89974de16, b51db75b-b22c-466f-8475-26c2ea99b2dc, b51db75b-b22c-466f-8475-26c2ea99b2dc,
  24b267fc-ab92-4a71-93c9-9a84785926cb, 24b267fc-ab92-4a71-93c9-9a84785926cb, b11c9ae9-c10b-4aef-a299-121d58515840,
  dfd95cd4-3009-4d13-8587-cb2658db9888, 604067cb-d2f5-47b5-bfa5-8c1e6021b9c8, 604067cb-d2f5-47b5-bfa5-8c1e6021b9c8,
  f4712b59-dd34-47ab-bc20-4965f9b36e2a, f4712b59-dd34-47ab-bc20-4965f9b36e2a, 8882f309-f497-40df-8cae-b358f53f1524,
  8882f309-f497-40df-8cae-b358f53f1524, ce504224-af83-4d89-be6d-f59d6f114597, ce504224-af83-4d89-be6d-f59d6f114597,
  6028db74-70af-40de-b62b-fc0c113f07ba, f1b3ba92-3bba-47ea-9e8f-a248057bc296, c48e49be-1f97-4218-9870-3b73490945b0,
  c48e49be-1f97-4218-9870-3b73490945b0, ade7bff4-1c5f-4009-8f3c-9d35c57a9936, ade7bff4-1c5f-4009-8f3c-9d35c57a9936,
  f2c90bdc-8325-446a-ae8a-94f1f0bce245, f2c90bdc-8325-446a-ae8a-94f1f0bce245, f63c03ca-f261-4c80-be3a-64f914e1e714,
  f63c03ca-f261-4c80-be3a-64f914e1e714, c10a9f4a-e6ae-4ae7-8eb5-7c2acb09dd89, 173d07bb-2512-4ec0-87e8-b4ba588b2781,
  7dde7e83-e1a7-4e8b-9675-ef7577bdcb86, 7dde7e83-e1a7-4e8b-9675-ef7577bdcb86, 2f529d61-3452-4472-9d6a-4904f61aaea2,
  2f529d61-3452-4472-9d6a-4904f61aaea2, 39ff8173-1e10-488f-99f4-6bac128918b5, 39ff8173-1e10-488f-99f4-6bac128918b5,
  ef72e9b0-f87b-4ba8-af44-1c3ac8613258, ef72e9b0-f87b-4ba8-af44-1c3ac8613258, 44892486-e918-44c3-b42c-9771db99aa8c,
  5abb38f3-a5ce-48c5-a1d2-609ffb791a69, c41a6308-4f6f-4073-871e-91b874d5c81f, c41a6308-4f6f-4073-871e-91b874d5c81f,
  49f57e6d-2474-4937-be38-f395aeaf21aa, 49f57e6d-2474-4937-be38-f395aeaf21aa, e48dae03-49a4-4f7d-9ab8-1e6184f6a575,
  e48dae03-49a4-4f7d-9ab8-1e6184f6a575, b26ededc-2ac5-47b3-928f-3f93f3ae3a4e, b26ededc-2ac5-47b3-928f-3f93f3ae3a4e,
  8364bb0f-00c3-40ed-855c-9033dc960649, c5bdffb1-2e50-42d0-a5be-3f3a4023e7c4, 42c3a87a-68d9-45df-8e4e-091a469b22cb,
  2f31e597-7c8a-4cdf-a4f6-113686a031df, a44f64b3-27c6-4436-a3a2-b49bb07cd974, a44f64b3-27c6-4436-a3a2-b49bb07cd974,
  2fe86d4c-83a7-4180-9618-a6524ebd2511, 2fe86d4c-83a7-4180-9618-a6524ebd2511, 91f1a197-f7b8-4653-9eb3-4c1a52ee21fb,
  4733f7f7-08bc-4678-86f2-e750edc0a510, 4356e371-d349-479b-a9c1-abb22346a6fc, 4356e371-d349-479b-a9c1-abb22346a6fc,
  2189297d-90fa-435f-acd1-63f7af2b5f6b, 2189297d-90fa-435f-acd1-63f7af2b5f6b, 690f55e0-87f6-4c5d-8a56-b4fe367ca62b,
  8bd310ba-9a12-4c5a-8f57-3bef729ed76e, c63be3e6-6a3f-4621-a118-0a216b7c0e0b, c63be3e6-6a3f-4621-a118-0a216b7c0e0b,
  44333768-5fa7-4dad-a870-f7be07f0afa0, 44333768-5fa7-4dad-a870-f7be07f0afa0, 8ff3b6db-241b-4c9a-9b3f-d436fc8b79c5,
  eb5f0148-97ba-45b5-a7a9-d7d0cbdfae26, 766478dd-f712-404a-b374-f0f85796719b, 766478dd-f712-404a-b374-f0f85796719b,
  5c9e1983-243e-4624-9c3c-a0f52932f22f, 5c9e1983-243e-4624-9c3c-a0f52932f22f, f96bef75-78fb-41ba-8f75-f5f8f5db2adf,
  f96bef75-78fb-41ba-8f75-f5f8f5db2adf, 91a2b1da-ec32-475e-b9b6-855972638751, 91a2b1da-ec32-475e-b9b6-855972638751,
  9024fce9-eb5d-48b4-9e70-16a693b8fd32, d7776ffe-ed0f-4368-9434-e7492b65d5b7, 6629a3d6-14a1-4ab7-a6a0-d3bd7f7303e1,
  48ce49bb-1940-4380-b22a-9579e2cd5169, 905e775f-8f6b-4303-84ef-f791ccecb448, 905e775f-8f6b-4303-84ef-f791ccecb448,
  5db5eb57-df39-471a-a950-8b27b2b9963c, 5db5eb57-df39-471a-a950-8b27b2b9963c, ced05dba-4c4d-430e-ae1e-4aa1d979843b,
  d012a1f1-5c07-41c8-a446-4a469898c6a3, 66fc4cee-84f2-42e4-aa7f-3986f49f2c60, 66fc4cee-84f2-42e4-aa7f-3986f49f2c60,
  64d71e60-591c-4002-9cfa-53502082eaac, 64d71e60-591c-4002-9cfa-53502082eaac, 9716ef04-9d33-4874-adfd-fd6867ab6cfb,
  ea0728f5-e93c-48c1-976e-54c262d5d8fa, f8019862-df9e-492f-86cf-ab012cd47fa2, f8019862-df9e-492f-86cf-ab012cd47fa2,
  6f0c1703-4408-4ddd-9bd4-e74d0d57334c, 6f0c1703-4408-4ddd-9bd4-e74d0d57334c, 56cf9ba0-4cf3-4957-b84b-39637fd5e2ae,
  9e7ace98-48ac-4da1-bdcb-311851d578a7, fb832fba-328c-4024-aa62-116b1cb6c6da, fb832fba-328c-4024-aa62-116b1cb6c6da,
  03ea5b5d-ad04-4e9f-a50b-a722057eb7df, 03ea5b5d-ad04-4e9f-a50b-a722057eb7df, 9dc00c84-fcb1-499b-b734-91ee5a295973,
  9dc00c84-fcb1-499b-b734-91ee5a295973, e723997f-6aa3-4868-a71b-56aaed817bca, 271bc16c-85c5-40b7-b4f6-1208d569b08d,
  cbf79ff4-d961-4379-9fc2-0d379b517bd1, e1874ac4-61b4-4238-b1da-6badb3b508ad, 05273595-b5b9-4bd8-8b72-8122d9b79ac8,
  b5ef1bf2-fcea-42d4-bc9e-9f6c6d921ab1, e279f55f-b8d1-4fca-9803-094972f97125, 6405de78-cddd-4fe7-aab1-df11059726e5,
  061eb5fc-c6a5-4575-98b8-15e0710af34d, 061eb5fc-c6a5-4575-98b8-15e0710af34d, 4f39ec35-9f6e-421b-94c9-91e2de32015d,
  4f39ec35-9f6e-421b-94c9-91e2de32015d, 4c993324-f59f-4c1e-9e32-8e63b3325479, 4c993324-f59f-4c1e-9e32-8e63b3325479,
  18a90346-b758-492c-b0c2-921e0e5176e7, 18a90346-b758-492c-b0c2-921e0e5176e7, 3fabf6f5-646c-4019-b7b8-043f2b6440f4,
  3fabf6f5-646c-4019-b7b8-043f2b6440f4, 3999dc32-0572-490e-9873-23259cf6a91e, cecce50b-ea2c-44da-bc7b-610cffa56172,
  cecce50b-ea2c-44da-bc7b-610cffa56172, 59fb1959-d979-4821-a376-beedab111400, 59fb1959-d979-4821-a376-beedab111400,
  42876f7b-5ab7-4809-92a8-95b1de18f99c, 42876f7b-5ab7-4809-92a8-95b1de18f99c, 3a464d12-91d7-4ed0-b88e-c556e088a821,
  7076e76d-fb57-4ab2-b0d7-81fff7e1b300, cb013a0a-92de-4741-8f2f-3b8a5fe87d9f, cb013a0a-92de-4741-8f2f-3b8a5fe87d9f,
  4a7ad181-4ec8-44aa-8e32-dce9784fb177, 4a7ad181-4ec8-44aa-8e32-dce9784fb177, fdc3777a-0bc3-48ff-ba11-bdde1de03140,
  3816520e-86b8-46b0-8507-a40f7ea81e5f, 68ed02f3-fc91-4e2a-9ea6-33920cd6f6f4, 68ed02f3-fc91-4e2a-9ea6-33920cd6f6f4,
  ee309bc1-acf2-4cd7-ba4c-66918dcac234, ee309bc1-acf2-4cd7-ba4c-66918dcac234, 2ba32c76-4f47-4952-9932-f1174f305fd3,
  ef157fc4-fad2-461e-90a0-8bd11820a669, e078a718-52af-47b3-b9d8-9fc22fc0fae8, e078a718-52af-47b3-b9d8-9fc22fc0fae8,
  ded0cd7a-61e6-4d23-ac18-3f5f9d5e0a3d, ded0cd7a-61e6-4d23-ac18-3f5f9d5e0a3d, 5734df8b-09a6-4f7a-b0dd-db0fcf844e8c,
  6110350e-34a6-4155-8bfa-ebdb10e18c33, 5759024e-ede9-4619-a43a-3aa722fac84b, 0ada6574-13be-4b75-97ed-16feae4a911c,
  7de556b0-5050-4946-8811-a35eaf890c12]
uuid_dest: [b0605c16-a416-4afc-b3c2-821e4fd9d7e5, 706b0231-edbb-443e-b80b-330691cdba41, 0d9f364e-901b-4845-8d6f-76c1122b97d1,
  23f0fa05-6629-4f0c-a27f-9505006c5fe2, 4dadd7bd-9471-4c57-9b42-9fa89974de16, 98aad7a8-2820-4b16-b15d-603f83e1a5da,
  b51db75b-b22c-466f-8475-26c2ea99b2dc, a0391f1b-8439-4dcd-b06b-3cbb3f2d7dfd, 24b267fc-ab92-4a71-93c9-9a84785926cb,
  a378fbe4-504f-4132-9dd8-68507765de36, b11c9ae9-c10b-4aef-a299-121d58515840, 0d9f364e-901b-4845-8d6f-76c1122b97d1,
  604067cb-d2f5-47b5-bfa5-8c1e6021b9c8, 4dadd7bd-9471-4c57-9b42-9fa89974de16, f4712b59-dd34-47ab-bc20-4965f9b36e2a,
  b51db75b-b22c-466f-8475-26c2ea99b2dc, 8882f309-f497-40df-8cae-b358f53f1524, 24b267fc-ab92-4a71-93c9-9a84785926cb,
  ce504224-af83-4d89-be6d-f59d6f114597, b11c9ae9-c10b-4aef-a299-121d58515840, 6028db74-70af-40de-b62b-fc0c113f07ba,
  f1b3ba92-3bba-47ea-9e8f-a248057bc296, c48e49be-1f97-4218-9870-3b73490945b0, f4712b59-dd34-47ab-bc20-4965f9b36e2a,
  ade7bff4-1c5f-4009-8f3c-9d35c57a9936, 8882f309-f497-40df-8cae-b358f53f1524, f2c90bdc-8325-446a-ae8a-94f1f0bce245,
  ce504224-af83-4d89-be6d-f59d6f114597, f63c03ca-f261-4c80-be3a-64f914e1e714, 6028db74-70af-40de-b62b-fc0c113f07ba,
  c10a9f4a-e6ae-4ae7-8eb5-7c2acb09dd89, 173d07bb-2512-4ec0-87e8-b4ba588b2781, 7dde7e83-e1a7-4e8b-9675-ef7577bdcb86,
  ade7bff4-1c5f-4009-8f3c-9d35c57a9936, 2f529d61-3452-4472-9d6a-4904f61aaea2, f2c90bdc-8325-446a-ae8a-94f1f0bce245,
  39ff8173-1e10-488f-99f4-6bac128918b5, f63c03ca-f261-4c80-be3a-64f914e1e714, ef72e9b0-f87b-4ba8-af44-1c3ac8613258,
  c10a9f4a-e6ae-4ae7-8eb5-7c2acb09dd89, 44892486-e918-44c3-b42c-9771db99aa8c, 5abb38f3-a5ce-48c5-a1d2-609ffb791a69,
  c41a6308-4f6f-4073-871e-91b874d5c81f, 2f529d61-3452-4472-9d6a-4904f61aaea2, 49f57e6d-2474-4937-be38-f395aeaf21aa,
  39ff8173-1e10-488f-99f4-6bac128918b5, e48dae03-49a4-4f7d-9ab8-1e6184f6a575, ef72e9b0-f87b-4ba8-af44-1c3ac8613258,
  b26ededc-2ac5-47b3-928f-3f93f3ae3a4e, 44892486-e918-44c3-b42c-9771db99aa8c, 8364bb0f-00c3-40ed-855c-9033dc960649,
  c5bdffb1-2e50-42d0-a5be-3f3a4023e7c4, 42c3a87a-68d9-45df-8e4e-091a469b22cb, 49f57e6d-2474-4937-be38-f395aeaf21aa,
  2f31e597-7c8a-4cdf-a4f6-113686a031df, e48dae03-49a4-4f7d-9ab8-1e6184f6a575, a44f64b3-27c6-4436-a3a2-b49bb07cd974,
  b26ededc-2ac5-47b3-928f-3f93f3ae3a4e, 2fe86d4c-83a7-4180-9618-a6524ebd2511, 8364bb0f-00c3-40ed-855c-9033dc960649,
  91f1a197-f7b8-4653-9eb3-4c1a52ee21fb, 4733f7f7-08bc-4678-86f2-e750edc0a510, 2f31e597-7c8a-4cdf-a4f6-113686a031df,
  a44f64b3-27c6-4436-a3a2-b49bb07cd974, 4356e371-d349-479b-a9c1-abb22346a6fc, 2fe86d4c-83a7-4180-9618-a6524ebd2511,
  2189297d-90fa-435f-acd1-63f7af2b5f6b, 91f1a197-f7b8-4653-9eb3-4c1a52ee21fb, 690f55e0-87f6-4c5d-8a56-b4fe367ca62b,
  8bd310ba-9a12-4c5a-8f57-3bef729ed76e, c63be3e6-6a3f-4621-a118-0a216b7c0e0b, 2189297d-90fa-435f-acd1-63f7af2b5f6b,
  44333768-5fa7-4dad-a870-f7be07f0afa0, 690f55e0-87f6-4c5d-8a56-b4fe367ca62b, 8ff3b6db-241b-4c9a-9b3f-d436fc8b79c5,
  eb5f0148-97ba-45b5-a7a9-d7d0cbdfae26, f96bef75-78fb-41ba-8f75-f5f8f5db2adf, 44333768-5fa7-4dad-a870-f7be07f0afa0,
  91a2b1da-ec32-475e-b9b6-855972638751, 8ff3b6db-241b-4c9a-9b3f-d436fc8b79c5, 9024fce9-eb5d-48b4-9e70-16a693b8fd32,
  d7776ffe-ed0f-4368-9434-e7492b65d5b7, 6629a3d6-14a1-4ab7-a6a0-d3bd7f7303e1, 5c9e1983-243e-4624-9c3c-a0f52932f22f,
  48ce49bb-1940-4380-b22a-9579e2cd5169, f96bef75-78fb-41ba-8f75-f5f8f5db2adf, 905e775f-8f6b-4303-84ef-f791ccecb448,
  91a2b1da-ec32-475e-b9b6-855972638751, 5db5eb57-df39-471a-a950-8b27b2b9963c, 9024fce9-eb5d-48b4-9e70-16a693b8fd32,
  ced05dba-4c4d-430e-ae1e-4aa1d979843b, d012a1f1-5c07-41c8-a446-4a469898c6a3, 48ce49bb-1940-4380-b22a-9579e2cd5169,
  905e775f-8f6b-4303-84ef-f791ccecb448, 66fc4cee-84f2-42e4-aa7f-3986f49f2c60, 5db5eb57-df39-471a-a950-8b27b2b9963c,
  64d71e60-591c-4002-9cfa-53502082eaac, ced05dba-4c4d-430e-ae1e-4aa1d979843b, 9716ef04-9d33-4874-adfd-fd6867ab6cfb,
  ea0728f5-e93c-48c1-976e-54c262d5d8fa, f8019862-df9e-492f-86cf-ab012cd47fa2, 64d71e60-591c-4002-9cfa-53502082eaac,
  6f0c1703-4408-4ddd-9bd4-e74d0d57334c, 9716ef04-9d33-4874-adfd-fd6867ab6cfb, 56cf9ba0-4cf3-4957-b84b-39637fd5e2ae,
  9e7ace98-48ac-4da1-bdcb-311851d578a7, 9dc00c84-fcb1-499b-b734-91ee5a295973, 6f0c1703-4408-4ddd-9bd4-e74d0d57334c,
  e723997f-6aa3-4868-a71b-56aaed817bca, 56cf9ba0-4cf3-4957-b84b-39637fd5e2ae, 93be216d-9b42-44b7-8103-5807bd34df42,
  271bc16c-85c5-40b7-b4f6-1208d569b08d, cbf79ff4-d961-4379-9fc2-0d379b517bd1, 03ea5b5d-ad04-4e9f-a50b-a722057eb7df,
  e1874ac4-61b4-4238-b1da-6badb3b508ad, 9dc00c84-fcb1-499b-b734-91ee5a295973, 05273595-b5b9-4bd8-8b72-8122d9b79ac8,
  e723997f-6aa3-4868-a71b-56aaed817bca, 93be216d-9b42-44b7-8103-5807bd34df42, b5ef1bf2-fcea-42d4-bc9e-9f6c6d921ab1,
  e1874ac4-61b4-4238-b1da-6badb3b508ad, 05273595-b5b9-4bd8-8b72-8122d9b79ac8, e279f55f-b8d1-4fca-9803-094972f97125,
  6405de78-cddd-4fe7-aab1-df11059726e5, 061eb5fc-c6a5-4575-98b8-15e0710af34d, 69894ffd-c7cb-4e92-9423-8c9f66101741,
  3fabf6f5-646c-4019-b7b8-043f2b6440f4, 4f39ec35-9f6e-421b-94c9-91e2de32015d, 3999dc32-0572-490e-9873-23259cf6a91e,
  22b4c79a-27e3-4c8e-978c-c8f85c5dd695, 59fb1959-d979-4821-a376-beedab111400, 18a90346-b758-492c-b0c2-921e0e5176e7,
  42876f7b-5ab7-4809-92a8-95b1de18f99c, 3fabf6f5-646c-4019-b7b8-043f2b6440f4, 3a464d12-91d7-4ed0-b88e-c556e088a821,
  3999dc32-0572-490e-9873-23259cf6a91e, d117f425-6c6e-478e-b9a0-a2c71754b2da, cb013a0a-92de-4741-8f2f-3b8a5fe87d9f,
  59fb1959-d979-4821-a376-beedab111400, 4a7ad181-4ec8-44aa-8e32-dce9784fb177, 42876f7b-5ab7-4809-92a8-95b1de18f99c,
  fdc3777a-0bc3-48ff-ba11-bdde1de03140, 3a464d12-91d7-4ed0-b88e-c556e088a821, d117f425-6c6e-478e-b9a0-a2c71754b2da,
3816520e-86b8-46b0-8507-a40f7ea81e5f, 68ed02f3-fc91-4e2a-9ea6-33920cd6f6f4, 4a7ad181-4ec8-44aa-8e32-dce9784fb177,
  ee309bc1-acf2-4cd7-ba4c-66918dcac234, fdc3777a-0bc3-48ff-ba11-bdde1de03140, 2ba32c76-4f47-4952-9932-f1174f305fd3,
  ef157fc4-fad2-461e-90a0-8bd11820a669, e078a718-52af-47b3-b9d8-9fc22fc0fae8, ee309bc1-acf2-4cd7-ba4c-66918dcac234,
  ded0cd7a-61e6-4d23-ac18-3f5f9d5e0a3d, 2ba32c76-4f47-4952-9932-f1174f305fd3, 5734df8b-09a6-4f7a-b0dd-db0fcf844e8c,
  6110350e-34a6-4155-8bfa-ebdb10e18c33, 5759024e-ede9-4619-a43a-3aa722fac84b, ded0cd7a-61e6-4d23-ac18-3f5f9d5e0a3d,
  0ada6574-13be-4b75-97ed-16feae4a911c, 5734df8b-09a6-4f7a-b0dd-db0fcf844e8c, 45f31b84-a6a3-47d1-a675-99aa1530827c,
  7de556b0-5050-4946-8811-a35eaf890c12, 0ada6574-13be-4b75-97ed-16feae4a911c, 45f31b84-a6a3-47d1-a675-99aa1530827c,
  d6f4983d-6653-4709-a772-48f81a8bb1b3]
name: [edge_10_19, edge_10_11, edge_11_20, edge_11_12, edge_12_21, edge_12_13, edge_13_22,
  edge_13_14, edge_14_23, edge_14_15, edge_15_24, edge_19_20, edge_20_29, edge_20_21,
  edge_21_30, edge_21_22, edge_22_31, edge_22_23, edge_23_32, edge_23_24, edge_24_33,
  edge_26_35, edge_29_38, edge_29_30, edge_30_39, edge_30_31, edge_31_40, edge_31_32,
  edge_32_41, edge_32_33, edge_33_42, edge_35_44, edge_38_47, edge_38_39, edge_39_48,
  edge_39_40, edge_40_49, edge_40_41, edge_41_50, edge_41_42, edge_42_51, edge_44_53,
  edge_47_56, edge_47_48, edge_48_57, edge_48_49, edge_49_58, edge_49_50, edge_50_59,
  edge_50_51, edge_51_60, edge_53_62, edge_56_65, edge_56_57, edge_57_66, edge_57_58,
  edge_58_67, edge_58_59, edge_59_68, edge_59_60, edge_60_69, edge_62_71, edge_65_66,
  edge_66_67, edge_67_76, edge_67_68, edge_68_77, edge_68_69, edge_69_78, edge_71_80,
  edge_76_85, edge_76_77, edge_77_86, edge_77_78, edge_78_87, edge_80_89, edge_85_94,
  edge_85_86, edge_86_95, edge_86_87, edge_87_96, edge_89_98, edge_92_101, edge_92_93,
  edge_93_102, edge_93_94, edge_94_103, edge_94_95, edge_95_104, edge_95_96, edge_96_105,
  edge_98_107, edge_101_102, edge_102_103, edge_103_112, edge_103_104, edge_104_113,
  edge_104_105, edge_105_114, edge_107_116, edge_112_121, edge_112_113, edge_113_122,
  edge_113_114, edge_114_123, edge_116_125, edge_121_130, edge_121_122, edge_122_131,
  edge_122_123, edge_123_132, edge_125_134, edge_128_137, edge_128_129, edge_129_138,
  edge_129_130, edge_130_139, edge_130_131, edge_131_132, edge_134_143, edge_137_138,
  edge_138_139, edge_139_148, edge_143_152, edge_148_157, edge_152_161, edge_157_166,
  edge_157_158, edge_158_167, edge_158_159, edge_164_173, edge_164_165, edge_165_174,
  edge_165_166, edge_166_175, edge_166_167, edge_167_176, edge_172_181, edge_172_173,
  edge_173_182, edge_173_174, edge_174_183, edge_174_175, edge_175_176, edge_179_188,
  edge_181_190, edge_181_182, edge_182_191, edge_182_183, edge_183_192, edge_188_197,
  edge_190_199, edge_190_191, edge_191_200, edge_191_192, edge_192_201, edge_197_206,
  edge_199_208, edge_199_200, edge_200_209, edge_200_201, edge_201_210, edge_206_215,
  edge_208_209, edge_209_210, edge_215_224]
uuid: [35051bf1-1f7d-4748-b2c2-72002a9bcd0a, 81848666-9bdf-480d-a44c-a0219d231800, fd05cca2-6917-4593-9d20-74063ca68a08,
  676234dd-4941-4122-8b97-422a8696c880, 93c368d7-e655-4d99-8a43-7a6dcdecff4b, 90ae4997-5047-4242-991b-9eb8f4c9b5f1,
  ca6662ad-201f-4e58-94b3-63c3040be554, 8e38beb4-5421-42b0-a92e-be1eb3338f2e, 08cb4d85-8255-44bd-8f7a-06256dd0c294,
  95417ce5-b761-4490-8b00-7ccf1481228e, 89f0063d-0857-4462-85ea-e78def4084ca, 9f996f67-757a-4613-908e-86a43168b0ac,
  85586285-4a78-4339-b030-d444b6b27126, 39d3ad5e-bb58-487e-a1aa-26de9cf96b7f, 06a6c9ba-d7bd-45a1-b938-242a0fb69342,
  f467da5d-e77a-4710-ad62-68a8a3579e15, b21cb64c-ff33-4ab8-95e6-92f1e9aa81ba, 96ddb368-07ad-4f7c-ade0-c799b75f8762,
  0f6faaae-f045-49a3-ba3a-191bad253b4b, c8449bd0-85b3-4262-bb2f-a4187f31bf3e, f6c497e9-cd54-46bd-b5e9-f197f798e31a,
  3c79407a-d560-4f21-99dc-8b39ac8016e4, 03b3f962-6efd-40ff-b1ce-4732ffb8a832, 81f5ec28-7d42-4539-98d9-a54c105ff669,
  7a32de79-7485-4b6b-b044-09257989429c, 1262acda-54cc-4758-9dd5-335003609913, 59ad915b-c589-44ef-a155-4f9139f89da9,
  40e363ea-b1d5-4614-b0f6-7ec7d2493e93, 2b4c9fc5-af2d-4d05-94bd-548be28cd374, a63b0ed5-1ce8-468d-ba5a-b493b34177e5,
  cc3243d6-9070-4a77-a659-0c77964d991f, 29f138de-2d19-49d6-b1be-8c5b51736f0b, 9cbcaea3-e64c-4794-9196-443da2d03f4b,
  48de549b-1086-4388-ad03-ab8af152659b, 5b53f64f-ac07-4b48-a496-151e31f66beb, 89ea6041-bee8-4e48-ac8e-81d8b128695d,
  45d040d3-0159-4bd0-9285-f7e61546f7d9, 2ad365ab-268d-4533-b024-61620692dbea, 74ba667b-42aa-43c0-b290-d776b8799e1d,
  2e788a7a-feea-4980-8e0c-cfe2b98573f7, 430b3b6d-41fa-4586-b77d-31342782b02f, d15512c7-a3fd-4d10-bf32-9961bbd97653,
  5f80bb7a-cb7f-4db2-8b08-f48e5fa0aa20, 5ebd74b1-dd91-47d4-b244-542f9d873a92, a0b8649c-8971-4ab0-a711-a3143d3834d4,
  c1fdb94c-7cc6-409a-bc21-3a037c10eb84, ae3e1c40-cc5a-426d-b178-30c744d8f091, 10a8c1cc-9527-4339-b361-f0e499af6226,
  6ace602f-1de0-4578-a13e-21d20ec7baf4, 76873ef6-027f-474d-b636-d8f478d70450, b97dc9ed-21a1-40e8-8b76-b8a396269019,
  62bdfe06-dc9d-4c51-8527-0ad0d31b4d32, fd80a0be-6abb-4eac-8e4b-8cfe22399b93, e79f9b8e-1960-421f-9853-8a0c321bbfa1,
  77b26897-fee3-4844-a787-e3cce73766a2, 80258cd4-fb8e-49ac-a9a5-c1aa35e54ce6, f27fc665-d0c4-4649-a4c6-5678c2a4ce58,
  9aa27b7a-cef0-4ad9-8065-cfede9412829, a1dbde49-dcfa-43f9-8747-a28d474be2e7, ab252960-50b8-4f73-be3f-ff1d2321a9aa,
  ef22a78c-fba6-4f79-ac5a-b1c0a216b483, 2e0bf79f-44a5-4479-ae33-6adb671c97a5, 2e4e6845-15af-4e6c-bd78-74a090d1be68,
  16b41c09-6392-42d4-865c-dd15d7ad52cc, dc7a3155-41c5-4871-812f-e6c1ce577861, 67d9a5da-b80b-4b3b-a603-bb282e0190a8,
  292d922e-a3f8-4099-ab46-53a67375b4c8, f1fcc86f-e8b9-4368-a32d-a0841a48a350, d86666fa-fe32-475e-bc56-2b8c6aec1c1d,
  94033167-708f-41f9-8028-6c57b9433f58, 93e566b2-f4c1-43b1-83c1-ebe0e8b82edf, ec776f4f-d0de-40dc-9148-2043542569f4,
  1df25c52-acd3-47f8-91ae-cb18a8d51ac7, 4fb05070-07fe-4266-b68c-dcbb59a1cddb, b7542b90-60a8-4375-bf2d-f074dd67fd9a,
  a125cca3-58a9-47bc-8f98-6c782153a486, 5035cdc5-0c5e-4770-84db-c041f80c885f, 79891f2e-33f1-435c-836a-c2dd35ce959d,
  e6687c9f-dadd-429f-9b7d-374a0df2db78, ca5f1e34-2e10-4f39-b695-f617fc74ca8d, e70093c2-0ad7-4a5c-bf32-7cea7c71fc1a,
  c9d569fe-8ddd-4032-936b-7eb01d04ddc8, fa5bf47e-bbb0-43f9-908f-bfac700eb767, 9d481300-9b18-4a8d-bf79-3ae0c4ba548e,
  6c48af75-cf99-46b5-82bb-e3e4f52832cd, ddde6574-a9ad-48a4-8617-6c5eec95fd35, edebe688-a802-4e86-82ee-7699c5c7d0fb,
  4edc71d5-cdfe-4b63-9419-dc5a92e6a534, 1dddcda4-3e42-4bbd-a370-9ae59440c25a, f7a17366-9411-40f2-bcd9-5de406fb6c6d,
  80e6a11b-6897-418e-919d-93a24ba1fde9, d6cb5b16-6ab7-4500-b1e2-709d6adfbc9d, 1d3bcdc9-1da6-4a44-9a43-a5045e24587a,
  0b6f6ee2-5ea5-4901-9b38-c1ab49f035bf, 66f5bad9-440e-47db-8c6d-96b87845d7e7, 60d33f56-da95-4fc0-93b2-d8d9859e1501,
  87d9139a-19fb-4c75-83ac-3ed24aa3bff6, 9b00b1e8-9757-4ec4-8f3f-418dd9402d3e, fd07583d-988f-46b2-aa70-5f201a230da8,
  7b7d0ca3-8941-41d8-9c14-f0e56d8c453e, 9b3b194c-3797-409d-9fd0-118e47d3e60e, 66a04c77-7952-4b74-8487-23d10a0e56cb,
  179eedab-9c2f-40a4-8917-6804fc2e1e7f, 6d47b2cc-c9ed-4b04-8984-2c23652f2a9c, e46c6ac0-0a11-4c79-be08-86686a415131,
  fbcc2d91-9bf3-46db-81f1-63e7c9e2b791, 38627158-0a47-4fd2-b25c-1fa9f1a49744, 3855ccd6-346d-45bc-a203-3579c8fd036b,
  a0a3ab72-c9cd-4700-bf6a-8afd1b8f260f, ec28c427-938e-488e-91d7-0b2c13b57d2b, 47335975-03bc-44b0-bc96-32067032cff4,
  50732cf7-3372-46d9-b52b-a8796f959f81, 865f1113-eaa4-49e2-b486-6f21c0ef71a7, 63913c99-aad0-4e43-9bdc-a4dd4820f2ab,
  2172c31f-5612-4bfd-8268-83abf2f1d1b1, f29d5307-2438-4557-8bd6-9aae921a9478, 7a3be7ad-d64e-4fbc-a3b8-08b73865b6ce,
  b630d88b-6021-42e6-bdc9-247617b437e7, 8d1bd4dc-8eb3-4436-916e-02aa5c34f823, 8b04d68d-3f9d-40cf-a20d-20003b9c18f6,
  c246576c-e590-441c-9234-a3c1e487a7f7, ad08aae6-3724-4c6c-8d80-320d12be4793, 58f78ad0-250a-42fe-a1fe-f3b93a439e4b,
  48003598-8398-4a15-b9e9-af214313521a, 0ac92ab3-1f27-46ef-8f6b-cf3a75c63727, a759e8c9-9e84-455c-9c00-a16824682121,
  c27f364f-3b07-44ec-9b6d-d4507ff6bfa7, 230843b2-ec0a-4725-819a-1472f4d0ec6a, 95fd3338-4284-496b-bd29-2c42b9aafe93,
  532cc225-b925-4a91-b929-88ab7fe726c5, 30b21723-99bb-4bb5-8b3c-819e3c4cf9de, 00597eeb-fd2c-4a59-ac16-85ab66a69321,
  c5b7d170-e9a8-4dd4-a34d-166478e64632, 89f5dc92-09b1-4887-a39d-f21a73e357b7, 48925c29-848e-4adf-82e2-6031821772e7,
  be782dfb-23d1-40a2-8bb9-82a7d5f038cc, 0cb89563-6b4d-4c48-8737-a37537142f07, 19782fd2-9f28-48fe-8dbf-fbdfc050454c,
  e5d07871-e5cd-4057-88a0-ab4c80ba956c, 2fbea29c-50d2-4adc-8c88-c22983e84d9d, 7c8ba745-8bc2-4a2c-8b57-7e20d85cdd15,
  ee1eb5e5-19a5-4a36-9866-ec1e4b4e1462, c8966e76-67c1-4b97-a58d-4d73bc0b6713, 350ba8e7-172e-44dc-b2c8-888c3d0354c4,
  29853318-8134-442b-b45d-e4b013bfc2a8, 16b638eb-7b23-4bcf-b31c-312ca5b0ead1, 09359e40-3a7b-4401-b525-de7f4f2c5db2,
  76e9c026-0232-4790-a2e3-bea08e204903, d4c3e35f-c82a-416f-a0d0-5b602545a09a, 05f51ce9-d666-482f-a55e-c4079725a23b,
  064dd852-c597-47b8-91de-e2c0d67635c7, 8253ffb6-79f7-4784-8a94-741939c17ae8, ca4bcd6b-462c-4ae7-9520-9450fc8537b7,
  bde9c5db-d46e-4b9c-b2a4-03bca96fe24f, ebce7947-71ff-49ee-a77c-d9c7d70b924a, b7b1233a-348d-4bed-ae5d-17bf54bb573a,
  72ade897-d587-4f5c-b92d-25981fe26f65, 7cf757c0-69f3-408f-9089-8e33c8de6991, 40bea417-8610-4ed9-af7a-fe46b0da2731,
  35cddfc8-dfff-405b-b84b-6733ae162ea6, 19d6474c-a9a2-465e-bef7-2331f4c5ac30, 25a6af8b-2171-4490-97db-97a10b7ded0a,
  ffad7313-8d70-4b63-a0b4-4d16d53a8631, 8019a4f2-b816-4b14-a7c1-c9e7cb6e936b, 85a5d25d-1172-471d-97d9-c7c45e5ce0eb,
  0a89a43e-75db-4e1f-9a36-3d6d4829c51e]
```
* Example output for the IML map - Pose with covariance
```
$rostopic echo /robot_0/pose_channel 
header: 
  seq: 76
  stamp: 
    secs: 36
    nsecs: 200000000
  frame_id: "map"
pose: 
  pose: 
    position: 
      x: 15.9636536984
      y: 13.3448596722
      z: 0.0
    orientation: 
      x: 0.0
      y: 0.0
      z: -0.718399874794
      w: 0.695630375915
  covariance: [0.23681326809196435, -0.0016609806341989497, 0.0, 0.0, 0.0, 0.0, -0.0016609806341989497, 0.23012575436405314, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.06503792236623226]
```


