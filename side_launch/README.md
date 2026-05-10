# side_launch

Auxiliary ROS2 launch files that **are not robonix packages** but are
needed alongside `rbnx boot` for the current bring-up.

## `static_tf.launch.xml`

Static TF tree for the Ranger Mini until `system.soma` ships in
robonix v0.2 (URDF + robot_state_publisher).

```bash
# In a separate shell from `rbnx boot`:
ros2 launch side_launch/static_tf.launch.xml
```

**Edit the XYZ + RPY values inside the file before using it for real.**
The committed values are placeholders and will give you incorrect
mapping / navigation if used as-is.

When soma lands, delete this whole directory and add the URDF path
to `system.soma.urdf_path` in `robonix_manifest.yaml`.
