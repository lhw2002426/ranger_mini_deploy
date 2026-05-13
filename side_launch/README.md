# side_launch

Auxiliary ROS2 launch files that **are not robonix packages** but are
needed alongside `rbnx boot` for the current bring-up.

## `static_tf.launch.xml`

Static TF tree for the Ranger Mini until `system.soma` ships in
robonix v0.2 (URDF + robot_state_publisher).

It publishes two static edges that mapping / nav need but no other
component owns:

- `base_link → livox_frame`        — MID-360 mount
- `base_link → camera_435i_link`   — RealSense D435i body mount
                                     (note the `camera_435i` prefix —
                                     this matches the deploy's
                                     `camera_name: camera_435i`, **not**
                                     the bare `camera_link`)

`odom → base_link` is owned by `ranger_chassis_rbnx` and `map → odom`
by rtabmap; this file deliberately does not republish them.

```bash
# In a separate shell from `rbnx boot`:
ros2 launch side_launch/static_tf.launch.xml
```

The committed mount offsets are **placeholders derived from CAD**
(see comments inside the file). Override per-axis on the command
line to match your physical robot before relying on this for SLAM
or navigation:

```bash
ros2 launch side_launch/static_tf.launch.xml \
    lidar_x:=0.18 lidar_z:=0.42 \
    camera_x:=0.28 camera_z:=0.30
```

Mis-calibrated TF is the single most common reason rtabmap "thinks
the robot is jumping around" or nav goals land 30 cm off — measure
the actual mount before trusting the defaults.

When soma lands, delete this whole directory and add the URDF path
to `system.soma.urdf_path` in `robonix_manifest.yaml`.
