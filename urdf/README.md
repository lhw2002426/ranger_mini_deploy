# urdf

Placeholder. Drop the calibrated Ranger Mini URDF here once it's ready,
then point `system.soma.urdf_path` in `robonix_manifest.yaml` at it
(soma itself will land in robonix v0.2 — see `HANDOFF.md` §九).

Required frames (per the conventions in the project README):

- `base_link`               — chassis frame, ground projection of geometric centre, X forward, Y left, Z up
- `livox_frame`             — MID-360 lidar mount
- `camera_link`             — RealSense D435i body mount
- `camera_color_optical_frame` — published by `realsense2_camera`; the URDF only needs the `camera_link` edge

Until this URDF exists, the static-tf substitute lives at
`../side_launch/static_tf.launch.xml`.
