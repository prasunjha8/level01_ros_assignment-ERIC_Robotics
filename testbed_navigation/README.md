# testbed_navigation

## Purpose

Provides modular Nav2 map loading, AMCL localization, and autonomous navigation for the testbed robot in Gazebo Classic.

## Files

- `launch/map_loader.launch.py`: starts the static map server and its lifecycle manager.
- `launch/localization.launch.py`: starts AMCL and its lifecycle manager.
- `launch/navigation.launch.py`: starts the Nav2 planning, control, behavior, and behavior-tree navigation servers with a separate lifecycle manager.
- `config/amcl_params.yaml`: AMCL and localization parameters.
- `config/nav2_params.yaml`: planner, controller, behavior, costmap, and BT navigator parameters.

## Nodes and Plugins

- `nav2_map_server`: publishes the configured static map.
- `nav2_amcl`: uses `DifferentialMotionModel` and the `likelihood_field` laser model.
- `planner_server`: uses `nav2_navfn_planner/NavfnPlanner`.
- `controller_server`: uses DWB (`dwb_core::DWBLocalPlanner`).
- `behavior_server`: provides spin, backup, drive-on-heading, and wait behaviors.
- `bt_navigator`: runs the default `NavigateToPose` behavior tree.
- Three separate lifecycle managers control map loading, localization, and navigation independently for modularity.

## Run

Use the workspace's Cyclone DDS configuration and start each launch in a separate terminal, in this order:

```bash
source /opt/ros/humble/setup.bash
source ~/assignment_ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export ROS_DOMAIN_ID=37
export CYCLONEDDS_URI=file://$HOME/cyclonedds.xml
~/rkill.sh
```

```bash
ros2 launch testbed_bringup testbed_full_bringup.launch.py
ros2 launch testbed_navigation map_loader.launch.py
ros2 launch testbed_navigation localization.launch.py
ros2 launch testbed_navigation navigation.launch.py
```

## Test Results

Localization (robot stationary at spawn, after in-place spins):
- AMCL error 0.02 m at spawn, 0.17 m after a slow 360° spin, 0.26 m after a fast 360° spin.

Navigation (NavigateToPose via `ros2 action send_goal`, true pose from Gazebo):
| Goal | Result | Final pose | Error |
|---|---|---|---|
| (2.0, 5.0) same room | SUCCEEDED | (1.79, 4.93) | 0.22 m |
| (6.0, 7.0) same room | SUCCEEDED | (5.84, 6.88) | 0.20 m |
| (3.6, -2.0) through doorway | SUCCEEDED | (3.69, -1.89) | 0.14 m |

All within the 0.20 m xy goal tolerance plus controller stop distance. 0 TF "too old" errors.
Screenshots / video: see `media/`.

## Challenges

- Lab LAN: default DDS domain shared with other PCs caused cross-talk and network drops that killed AMCL's lifecycle bond. Fixed with Cyclone DDS bound to loopback (multicast enabled) and `ROS_DOMAIN_ID=37`.
- `ROS_LOCALHOST_ONLY` with Cyclone limits participants (~10) -> Nav2 nodes failed "free participant index". Replaced with a Cyclone XML config.
- `pkill -f ros2` does not kill node processes; leftover lifecycle managers caused map_server to hang unconfigured. Use `~/rkill.sh`, which matches `--ros-args`.
- Lifecycle bonds timed out under load -> set `bond_timeout: 0.0`.
