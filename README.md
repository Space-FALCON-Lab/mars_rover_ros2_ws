# Mars Rover ROS 2 Workspace

This workspace receives simulated rover measurements from Unity and sends
velocity commands back. It contains a ROS TCP Endpoint, an OctoMap launch,
a local costmap projector and a custom A* planner.

Use the [paired simulation repository](https://github.com/Space-FALCON-Lab/Mars_Rover_Navigation_Sim)
to obtain the corresponding Unity project. This guide covers the ROS side.
The planner is an experimental baseline with a source-defined goal; the launch
does not start Nav2.

## Before you start

The commands below use Bash in a ROS 2 Humble environment on Ubuntu. Install
ROS, colcon and rosdep using the
[ROS setup instructions](https://docs.ros.org/en/humble/Installation.html), then
initialize rosdep if this is its first use on the machine.

The nodes require OctoMap, RViz, TF2, NumPy, `tf_transformations` and the ROS
message packages used in their imports. The current package manifest omits
some of these Python/message dependencies, so the build instructions install
those explicitly as well as running rosdep. The ROS Index documents the
[`tf_transformations` binary package](https://index.ros.org/p/tf_transformations/).

## Build and run

If you obtained this workspace through the paired repository, open a terminal
in its `mars_rover_ros2_ws/` directory. To use this repository on its own:

```bash
git clone https://github.com/Space-FALCON-Lab/mars_rover_ros2_ws.git
cd mars_rover_ros2_ws
```

From that workspace directory:

```bash
source /opt/ros/humble/setup.bash
rosdep update
rosdep install --from-paths src --ignore-src --rosdistro humble -r -y
sudo apt install python3-numpy ros-humble-tf-transformations ros-humble-visualization-msgs ros-humble-std-srvs
colcon build --symlink-install
source install/setup.bash
python3 -c "import numpy, tf_transformations; from visualization_msgs.msg import MarkerArray; from std_srvs.srv import Empty"
```

The import check should finish without an error. colcon creates `build/`,
`install/` and `log/`; the
[colcon reference](https://colcon.readthedocs.io/en/released/reference/verb/build.html)
explains the symlink-install option.

Keep this terminal open for the ROS TCP Endpoint:

```bash
ros2 launch ros_tcp_endpoint endpoint.py
```

It listens on port `10000`. In a second terminal, return to the workspace
directory, source the same environments, then launch the mapping and planner:

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
cd src/mars_rover_sim
ros2 launch mars_rover_sim bringup_sim.launch.py
```

The last `cd` is required by the current launch file: RViz opens
`config/octomap.rviz` relative to the terminal's working directory. The launch
starts OctoMap, the costmap projector, the custom planner and RViz. The TCP
Endpoint is a separate process.

Configure Unity's ROS connection for ROS 2, the ROS computer's address and
port `10000`, then enter Play mode in the paired scene.

## Check the connection

In another terminal, source ROS and the workspace from its root before using
these commands:

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 topic list
ros2 topic echo /clock --once
ros2 run tf2_ros tf2_echo map base_link
```

The clock should advance while Unity is playing, and the transform check should
report the rover pose. The LiDAR frame in each point-cloud header must also
connect to `map`; the supplied Unity publisher defaults to `lidar_link`.

| Topic | Direction or purpose |
| --- | --- |
| `/pointcloud` | Unity point cloud; remapped into OctoMap's cloud input |
| `/clock` | Unity simulated time |
| `/tf`, `/tf_static` | Transforms needed by mapping and planning |
| `/occupied_cells_vis_array` | OctoMap markers consumed by the projector |
| `/costmap_2d` | Projected `OccupancyGrid` |
| `/astar_path` | The planner's selected `Path` |
| `/cmd_vel` | `Twist` commands sent to the Unity rover |

The launch does not enable its commented-out transform publishers. If your
scene supplies `/pose` but not `map` to `base_link`, the supplied
`scripts/pose_to_tf.py` is an alternative broadcaster. Run it only when that
transform has no other publisher, after confirming that the pose is expressed
in `map`. It does not supply the separate LiDAR mounting transform.

## Understand and change the experiment

The costmap projector groups points by horizontal cell and compares their
vertical spread. Its `slope_threshold` is a height difference in metres,
despite the parameter's name; it is not an angle. It uses a local window around
the rover and requests an OctoMap reset periodically.

The planner sets `goal_dx = -1000.0` and `goal_dy = 0.0` relative to the first
rover pose. Change those fields in
[`simple_astar_navigator.py`](src/mars_rover_sim/mars_rover_sim/simple_astar_navigator.py)
for a different source-defined goal. It does not subscribe to an RViz goal
topic.

Occupied cells add a finite path cost rather than being forbidden, and unknown
cells receive no additional cost. If the inflated-grid search fails, the
planner tries the raw grid. A published path therefore does not establish
collision avoidance or terrain traversability.

There is also a current failure case: if the first searches find no path,
the planner can access `last_good_path` before it has been set. That needs a
code repair before relying on the no-path behavior. The setup steps above do
not repair it.

## If a stage produces no output

- **Package or executable not found:** build successfully and source
  `install/setup.bash` in that terminal.
- **Missing Python module:** complete the explicit dependency step and repeat
  the import check.
- **No point cloud or clock:** check Unity Play mode, its ROS connection
  address and the TCP Endpoint process.
- **Transform lookup failed:** check the `map`, `base_link` and point-cloud
  frame chain; the launch does not create it for you.
- **RViz configuration not found:** launch from `src/mars_rover_sim/`, as above.
- **Costmap but no path:** inspect the goal and the planner's no-path messages,
  including the current first-failure issue described above.

The vendored [ROS TCP Endpoint documentation](src/ros_tcp_endpoint/README.md)
retains its upstream instructions and attribution.

## Contributors

- [Giusy Falcone](https://github.com/gfalcon2)
- [Maxwell Gonzalez](https://github.com/Maxwell44772029)

Upstream ROS TCP Endpoint attribution remains in its vendored documentation and license.
