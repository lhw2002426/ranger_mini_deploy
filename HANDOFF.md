# Ranger Mini bring-up — handoff

你的任务是把 robonix v0.1.x 在 Jetson Orin + Ranger Mini 上**起来**：`rbnx build` 拉所有包、`rbnx boot` 跑起来，看 scene、底盘驱动、SLAM 这些能不能正常 active。语音 / liaison voice / soma（URDF）这一轮**不测**，已经从 manifest 里关掉了。

> 关于 boot 模型：rbnx boot **严格按 YAML 声明顺序串行**，没有 defer 队列、没有自动 retry。任意一个 `Driver(CMD_INIT)` 返回 `ok=false`（或 90s 超时）就直接 bail 整个部署。出错时不要"等等再试"，去看那一个包的 log。

## 目录在哪

Jetson 上 ssh 进去后：

```bash
ssh syswonder@<jetson-ip>
```

robonix 主仓在 `/home/syswonder/wheatfox/robonix`（包含 `rbnx` 工具链 + Python pylib + 系统组件）。

部署仓库（这个 repo）放到 `~/robonix-v0.1-deploy/`：

```bash
cd ~
git clone https://github.com/enkerewpo/ranger_mini_deploy robonix-v0.1-deploy
cd robonix-v0.1-deploy
```

## 〇、上电前置 checklist

跳过这一节后面**多半会卡在 build 或 boot**，先逐项确认：

- [ ] **rbnx 已装好**：`rbnx --version` 能输出版本号；`which rbnx` 在 `~/.cargo/bin/`。如果没有：
  ```bash
  cd /home/syswonder/wheatfox/robonix/rust && make install
  ```
- [ ] **robonix 源码路径已注册**：
  ```bash
  rbnx path root        # 应输出 /home/syswonder/wheatfox/robonix
  ```
  报错就重跑：`( cd /home/syswonder/wheatfox/robonix && rbnx setup )`
- [ ] **ROS2 环境已 source**：`echo $ROS_DISTRO` 输出 `humble`（或对应版本）。没有就先 `source /opt/ros/humble/setup.bash`。
- [ ] **CAN 接口起来了**：`ip link show can_ranger`，state 应是 `UP`。如未：
  ```bash
  sudo ip link set can_ranger type can bitrate 500000
  sudo ip link set can_ranger up
  ```
- [ ] **MID-360 lidar 通电、网线接通**：`ping 192.168.1.161` 通。
- [ ] **RealSense 插好 USB 3.0**：`lsusb | grep -i realsense` 见到 `Intel`。
- [ ] **VLM 凭据准备好**：复制并填好 `.env`（见下一步）。

## 一、配置 VLM 凭据

```bash
cp .env.example .env
# 用 nano/vim 把 VLM_BASE_URL / VLM_API_KEY / VLM_MODEL 三项填上真实值
# 凭据找 wheatfox 拿
```

每个 shell 进入部署目录后 `source` 一下：

```bash
set -a; source .env; set +a
```

或者把 `set -a; source ~/robonix-v0.1-deploy/.env; set +a` 放进 `~/.bashrc`。

## 二、validate（先静态查一下）

```bash
rbnx validate
```

manifest 写错（缩进、字段拼写、`${VAR}` 引用）会在这一步立刻报，不必等到 boot 才发现。

## 三、build

```bash
rbnx build
```

`rbnx build` 会：
1. clone manifest 里每个 `url:` 对应的包到 `rbnx-boot/cache/<repo-name>/`（mid360_lidar_rbnx / mid360_imu_rbnx / realsense_camera_rbnx / ranger_chassis_rbnx / mapping_rbnx / explore_rbnx）。
2. 给每个包跑它的 `scripts/build.sh`（colcon / pip / docker，看包）。
3. 给每个包做 codegen 生成 atlas + contracts 的 Python stubs。

build 跑完每个 cap 应当显示 `✓`。任何包失败先看 `rbnx-boot/logs/<pkg>-build.log`（具体文件名以 build 输出末尾的提示为准）。

## 四、改包

要改任何 url-fetched 的包（比如调 livox 参数 / 改 rtabmap 配置），**直接进 `rbnx-boot/cache/<pkg>/` 改文件**——那是个完整 git clone，可以正常 `git diff` / `git commit` / 自己 push。**不要重新 clone 工作树**——一次 `rbnx build` 后再改不会被覆盖。

如果改完发现要重 build 那个包：进 `rbnx-boot/cache/<pkg>/`，跑 `bash scripts/build.sh`。

## 五、boot

确认 `.env` 已 source 后：

```bash
rbnx boot
```

启动顺序（**严格按 manifest 声明顺序串行**，没有 defer 协议）：

1. **system**：atlas → executor → pilot → liaison → memory → scene
2. **primitive**：mid360_lidar → mid360_imu → realsense_camera → ranger_chassis
3. **service**：mapping → nav2
4. **skill**：explore（boot 后停在 `INACTIVE` 等 LLM 调用是正常的，executor 在第一次 tool call 时发 `Driver(CMD_ACTIVATE)`，前提是 explore 包声明并实现了 `*/driver` 合约）

任意一个 `Driver(CMD_INIT)` 返回 `ok=false` 或超时 90s，整个 boot 直接退出，不会自动 retry。

> ℹ️ **nav2 config 为空**：manifest 里 `nav2.config: {}`，走 wrapper 自带的默认值。如果它在 Init 阶段挂掉（常见于缺 `map_topic` / `robot_radius` / `footprint` 这类必填），先按 §十.9 审计 `on_init` 再补字段。

正常完成时输出底部应该是：

```
✓ N component(s) up; logs under .../rbnx-boot/logs
```

`rbnx boot` 会**阻塞**——保持这个终端不要关。

## 六、验证

新开一个终端（仍在部署目录里）：

### 6.1 atlas 状态

```bash
rbnx caps -v
```

期待看到所有 primitive / service / system 都是 `[ACTIVE]`，`explore` skill 是 `[INACTIVE]`（lazy-activate）。如果某个卡在 `[INACTIVE]` 或 `[ERROR]`：

```bash
tail -n 100 rbnx-boot/logs/<component>.log
```

### 6.2 contracts

```bash
rbnx contracts
```

应能看到 `robonix/primitive/{lidar,imu,camera,chassis}/*` + `robonix/service/{map,navigation}/*` + 各 system 服务的 contract。

### 6.3 LLM 可见的工具

```bash
rbnx tools
```

每行一个 MCP 工具，名字形如 `<area>_<leaf>`（如 `camera_snapshot` / `chassis_move`）。**没看到的工具 LLM 就调不到**——大多数情况是某个 cap 没起来或没声明对应的 MCP 接口。

### 6.4 scene WebUI

scene 起来会监听一个 HTTP 端口（看 `rbnx-boot/logs/system_scene.log` 里 `MCP HTTP serving on 0.0.0.0:XXXXX` 这一行）。浏览器打开看 2D / 3D / 摄像头视图能不能加载、对象能不能识别、底盘 pose 能不能更新。

### 6.5 chat（可选，端到端）

```bash
rbnx chat
```

走 liaison → pilot 链路，能不能问"导航到 X"之类的让 explore 被 LLM 路由 → activate → 跑起来。nav2 也在这一轮启用，所以确定性目标点（`navigate_to (x, y)` 类工具）应该可用。frontier explore 仍然可以独立跑。

### 6.6 单 cap 调试（独立测试）

某个包卡住时，**用一份 mini-manifest 跑 boot 路径**——这是与生产一致的 Driver(CMD_INIT) 流程：

```bash
cat > /tmp/mini.yaml <<'EOF'
manifestVersion: 1
name: ranger-debug-lidar
system:
  atlas:    { listen: 127.0.0.1:50051, log: info }
  executor: { listen: 127.0.0.1:50061, log: info }
  pilot:
    listen: 127.0.0.1:50071
    log: info
    vlm:
      upstream: ${VLM_BASE_URL}
      api_key:  ${VLM_API_KEY}
      model:    ${VLM_MODEL}
      api_format: openai
primitive:
  - name: mid360_lidar
    path: rbnx-boot/cache/mid360_lidar_rbnx
    config:
      lidar_topic: /scanner/cloud
      lidar_ip: 192.168.1.161
      xfer_format: 2
      sentinel_timeout_s: 30.0
EOF
rbnx boot -f /tmp/mini.yaml
```

> ⚠️ 不要用 `rbnx start -p ... -c <yaml>` 来调试单包——它把 config 通过 `RBNX_CONFIG_FILE` env 喂给 cap 进程，**走的不是** `Driver(CMD_INIT, config_json)` 的生产通道，行为可能不一致。

## 七、关闭

```
Ctrl-C        # 在 rbnx boot 终端；它会反向 teardown
```

如果遗留进程没清干净（`pgrep -f mid360\|realsense\|ranger\|rtabmap` 还有命中）：

```bash
rbnx shutdown    # 读 rbnx-boot/state.json 按 PGID kill
# 兜底：
pkill -f livox_ros_driver2
pkill -f realsense2_camera
pkill -f rtabmap
```

慎用 `pkill -f robonix-`——它会杀掉 system 那几个 Rust 二进制但**不会**杀掉用户 cap 的 Python 进程，留下端口僵尸。

## 八、清理

```bash
rbnx clean -f robonix_manifest.yaml          # 清所有包的 rbnx-build/ + rbnx-boot/{logs,state.json}
rbnx clean -f robonix_manifest.yaml --cache  # 同上，并 wipe rbnx-boot/cache/（强制下次重新 git clone）
```

## 九、当前已知关掉的功能

| 功能 | 状态 | 备注 |
|---|---|---|
| soma（URDF / robot_state_publisher）| ☐ | v0.2 roadmap，未实现；用 `side_launch/static_tf.launch.xml` 顶（先量好实际安装位移再 launch） |
| 语音 / speech 服务 | ☐ | 麦克风未接，audio primitives 也没列在 manifest |
| liaison voice loop | ☐ | 同上；但 liaison 自身**开**了，因为 `rbnx chat` 要走它 |
| nav2 | ⚠️ `config: {}` | manifest 里**启用**但 config 留空，走 wrapper 默认值；Init 挂了就按 §十.9 审 `on_init` 补字段 |
| fastlio2 SLAM | 已知 drift | 由 wheatfox 单独测试，**不走 rbnx boot**；当前 mapping 用 rtabmap 算法 |

## 十、卡住找谁

- build 报错 / API 不对 / state machine 逻辑：wheatfox
- 底盘 / lidar / IMU 硬件 / CAN：（你们组对应的硬件同学）
- VLM 凭据 / pilot 路由：wheatfox
- nav2 必填 config 字段：检查 `rbnx-boot/cache/nav2_wrapper_rbnx/<wherever>/on_init.py`，需要的字段直接读源码

build + boot 跑通后给一份 `rbnx caps -v` 的输出截图就是 bring-up 报告。
