# ranger_mini_deploy

Robonix deploy manifest for the AgileX Ranger Mini robot at SysWonder lab.

Hardware: Jetson Orin (aarch64, Tegra-special CUDA stack), AgileX Ranger Mini v2 chassis (CAN bus), Livox MID-360 3D LiDAR + integrated 6-axis IMU (Ethernet), Intel RealSense D435i RGBD camera (USB 3.0, with internal IMU).

## Packages

All package URLs in `robonix_manifest.yaml` resolve from this enkerewpo GitHub org:

| Package                  | Repo                                             | Owns                  |
| ------------------------ | ------------------------------------------------ | --------------------- |
| `mid360_lidar_rbnx`      | enkerewpo/mid360_lidar_rbnx                      | primitive/lidar/*     |
| `mid360_imu_rbnx`        | enkerewpo/mid360_imu_rbnx                        | primitive/imu/*       |
| `realsense_camera_rbnx`  | enkerewpo/realsense_camera_rbnx                  | primitive/camera/*    |
| `ranger_chassis_rbnx`    | enkerewpo/ranger_chassis_rbnx                    | primitive/chassis/*   |
| `mapping_rbnx`           | enkerewpo/mapping_rbnx                           | service/map/*         |
| `nav2_wrapper_rbnx`      | enkerewpo/nav2_wrapper_rbnx                      | service/navigation/*  |
| `explore_rbnx`           | enkerewpo/explore_rbnx                           | skill/explore/*       |

`nav2_wrapper_rbnx` ships with `config: {}` — the wrapper applies its own defaults. If `Driver(CMD_INIT)` on nav2 returns `ok=false` on first bring-up, audit its required fields (see HANDOFF §十.9) and fill them under the `config:` block in `robonix_manifest.yaml`.

## Quickstart

```bash
# On the Jetson, in this directory:
cp .env.example .env             # then edit .env to fill VLM_BASE_URL / VLM_API_KEY / VLM_MODEL
set -a; source .env; set +a      # export the vars into the current shell

rbnx validate                    # static check the manifest
rbnx build                       # clones each url: package and runs its build.sh
rbnx boot                        # spawns each one and runs Driver(CMD_INIT, config_json)
```

`rbnx build` writes everything under `rbnx-boot/cache/<repo-name>/`. Each
package's per-clone build artefacts go to `<pkg>/rbnx-build/`. The original
working dir on the Jetson is never touched.

`rbnx boot` blocks until you Ctrl-C; it tears down every spawned PGID in
reverse order on exit. Detached cleanup is in `rbnx shutdown`.

## URDF — required, not shipped

Soma (URDF + robot_state_publisher) needs a Ranger Mini URDF to publish the static TF tree. The URDF must include `base_link` (chassis frame; convention: ground projection of the geometric centre, X forward, Z up), `livox_frame` mount transform from `base_link`, and `camera_435i_link` + `camera_435i_color_optical_frame` mount transforms.

Until a calibrated URDF is in hand, the **`ranger_description` primitive** in the manifest stands in for soma: it spawns two `static_transform_publisher` nodes (`base_link → livox_frame` and `base_link → camera_435i_link`) at boot. The mount-offset defaults baked into its launch file are placeholders derived from CAD — measure your physical robot and override via the `launch_args` block under that primitive in `robonix_manifest.yaml`. Source lives at <https://github.com/lhw2002426/ranger_description_rbnx>.

When the URDF is ready, drop `ranger_description` from the manifest and add `system.soma.urdf_path` instead.

## Verifying the bring-up

After `rbnx boot` settles:

```bash
ros2 topic hz /scanner/cloud   # ~10 Hz lidar PointCloud2
ros2 topic hz /livox/imu       # ~200 Hz sensor_msgs/Imu
ros2 topic hz /camera_435i/color/image_raw                    # ~30 Hz
ros2 topic hz /camera_435i/aligned_depth_to_color/image_raw   # ~30 Hz
ros2 topic hz /odom            # chassis odometry (~50 Hz)
ros2 topic hz /map             # 1 Hz-ish OccupancyGrid (from rtabmap)
ros2 topic echo /robonix/map/pose --once

rbnx caps -v                   # all primitive/* + service/map/* + system/* should show [ACTIVE].
                               # explore (skill) is expected to be [INACTIVE] until first LLM call.
rbnx contracts                 # static schema view of every contract atlas loaded
rbnx tools                     # exact MCP tool list the pilot exposes to the LLM
```

Open RViz and load the rtabmap visualization config to see the map build up.

## Boot sequencing — actual mechanics

`rbnx boot` launches every package **strictly serially in YAML declaration order**: each `primitive[i]` must reach Driver(CMD_INIT) `ok=true` (within `DRIVER_INIT_TIMEOUT = 90s`) before `primitive[i+1]` is spawned; same for `service:` and `skill:`. There is **no** automatic retry — a package whose Driver returns `ok=false` aborts the entire boot. Source: `rust/crates/robonix-cli/src/cmd/deploy.rs` (`if !r.ok { bail }` near line 1430, `boot_section_serial` loop above it).

Concretely, the cascade for this stack:

```
mid360_lidar.Init     → spawns livox driver, declares lidar3d
                        (also makes /livox/imu live on the bus)
mid360_imu.Init       → subscribes /livox/imu, declares primitive/imu/*
                        (FAILS the boot if mid360_lidar didn't actually
                        bring /livox/imu up — there's no retry)
realsense_camera.Init → spawns realsense, declares rgb + depth
ranger_chassis.Init   → opens CAN, publishes /odom, declares chassis/*
mapping.Init          → queries atlas for lidar3d + rgb + depth + odom,
                        configures rtabmap to fuse them
nav2.Init             → queries atlas for service/map/occupancy_grid
                        (must see mapping's declaration — ordering matters)
explore               → registers; stays INACTIVE until the LLM picks
                        one of its tools (executor.dispatch sticky-activates)
```

If you need to reorder: edit `robonix_manifest.yaml`. The list **is** the dependency declaration; consumers must come after providers.

## Layout (after first boot)

```
ranger_mini_deploy/
├── robonix_manifest.yaml
├── README.md
├── HANDOFF.md
├── .env.example
├── .gitignore
├── urdf/
│   └── README.md
└── rbnx-boot/                  ← gitignored, auto-generated
    ├── cache/<pkg>/            ← git clone of every url: package
    │   └── rbnx-build/         ← per-clone build artefacts + sentinel
    ├── instances/<pkg>.json    ← per-package config snapshot (debug only)
    ├── logs/<component>.log    ← stdout+stderr of every spawned process
    └── state.json              ← PGID/PID list for `rbnx shutdown`
```

## License

Manifest + this README: MulanPSL-2.0. Each `url:` package retains its own license.
