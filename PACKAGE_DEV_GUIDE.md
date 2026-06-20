# Robonix Package 开发指南（基于 Ranger Mini 实战经验）

本文档面向 **想把自己的一个 ROS 2 节点 / 一段感知/控制逻辑 / 一个 launch 文件** 接到 Robonix 框架里的开发者。它把 `/Users/howenliu/lab/packages/` 里 8 个 ranger 系列包的设计模式 + 常见坑点 + Ranger Mini 部署里学到的经验，整理成可复用的工程套路。

> 框架本身的设计哲学（atlas/executor/pilot 三件套、能力命名空间、生命周期状态机等）请读 `/Users/howenliu/lab/robonix/CLAUDE.md`、`/Users/howenliu/lab/robonix/README.md` 和 `system/scene/README.md`。本文档只讲**怎么动手做一个包**。

---

## 0. 心智模型：robonix package 到底是什么

一个 Robonix package = **一个进程的入口** + **该进程在 atlas 上注册的若干能力（capability）声明**。

- `package_manifest.yaml` 是合同。它告诉 `rbnx build` 怎么编、`rbnx boot` 怎么启、要在 atlas 上声明哪些 capability。
- `scripts/start.sh` 是入口。它会被 `rbnx boot` 当成长期运行的子进程拉起，由 rbnx 监督生命周期（rbnx SIGTERM 时连同子进程一起干掉）。
- `scripts/build.sh` 在第一次 boot 时（或者 `rbnx build` 时）跑一次。常见做法是用 `rbnx codegen` 生成一些 atlas/MCP gRPC stub 给本包用。

启动后这个进程要做的核心三件事：
1. **向 atlas 注册自己**（一次 `RegisterPrimitive` / `RegisterService` / `RegisterSkill` RPC）
2. **声明 capability**（每个声明一次 `DeclareCapability`，告诉 atlas 这个 contract 走哪个 transport、endpoint 是啥）
3. **周期性 heartbeat**（默认 90 秒不发就被 atlas 标 TERMINATED；推荐 30 秒一次）

绝大多数情况下，上面 1+3 由 `robonix_api.Capability` 自动做（`Capability.bootstrap()` 一行）；2 要么由 `Capability.declare_*` 主动调，要么由 `@on_init` lifecycle 钩子在合适时机做。

---

## 1. 三类 package 模板：选哪个套？

把你要做的事按下面四个维度判断：

| 你的工作 | Driver(CMD_INIT) 是否必要 | 是否要暴露 atlas 路由的 topic / RPC | 选哪个模板 |
|----------|--------------------------|--------------------------------------|-----------|
| 启动一个 ROS launch、不参与任何 atlas 路由 | 否 | 否 | **模板 A：极简 launch wrapper** |
| 包装一个传感器/上游 ROS 节点，注册成 atlas 上的某个 contract（`primitive/lidar/lidar3d` 之类） | 是 | 是 | **模板 B：典型 primitive/service** |
| 暴露 LLM-callable MCP 工具（pilot 会通过 LLM 调用） | 是 | 是（transport=mcp） | **模板 C：带 MCP 的 service/skill** |

下面分别给参考实现和模板代码。

---

## 2. 模板 A：极简 launch wrapper

**典型场景**：你只想让 rbnx boot 拉起一个 `ros2 launch xxx.launch.xml`，不需要参与 atlas 路由，但又不想用 `tmux` / `systemd` 等其他生命周期管理。

**参考实现**：[`packages/ranger_description_rbnx`](../packages/ranger_description_rbnx/)。它就是把两个 `static_transform_publisher` 用 launch 文件包起来发布 base_link → 传感器静态 TF。

### 目录结构

```
my_package_rbnx/
├── package_manifest.yaml
├── launch/
│   └── my.launch.xml
└── scripts/
    ├── build.sh
    ├── start.sh
    └── atlas_register_and_launch.py
```

### `package_manifest.yaml`

```yaml
manifestVersion: 1

package:
  name: com.<vendor>.<role>.<my_package>
  version: 0.1.0
  vendor: <vendor>
  description: <一句话>
  license: MulanPSL-2.0

build: bash scripts/build.sh
start: bash scripts/start.sh

# 故意为空。我们注册一个 provider 到 atlas（让 rbnx boot 不卡），
# 但不暴露任何路由 contract — TF / launch 输出不需要走 atlas。
capabilities: []
```

**关键点**：`capabilities: []` 时 rbnx boot（`deploy.rs:1247-1253`）会直接把这个包当成 `ACTIVE (no driver)`，跳过 `Driver(CMD_INIT)` 握手。

### `scripts/build.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
PKG="${RBNX_PACKAGE_ROOT:-$(cd "$(dirname "$0")/.." && pwd)}"
cd "$PKG"

mkdir -p rbnx-build/data
FLAGS=(--out-dir "$PKG/rbnx-build/codegen")
[[ "${RBNX_BUILD_CLEAN:-}" == "1" ]] && FLAGS+=(--clean)
rbnx codegen -p "$PKG" "${FLAGS[@]}"     # 生成 atlas_pb2 让 start 时能 RegisterPrimitive

touch "$PKG/rbnx-build/.rbnx-built"
```

`rbnx codegen` 是必须的 —— 即使你不暴露 contract，也要拿到 `atlas_pb2.py` 才能注册到 atlas。

### `scripts/start.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
PKG="${RBNX_PACKAGE_ROOT:-$(cd "$(dirname "$0")/.." && pwd)}"
cd "$PKG"

ROS_DISTRO="${ROS_DISTRO:-humble}"
set +u; source "/opt/ros/${ROS_DISTRO}/setup.bash"; set -u

CODEGEN="$PKG/rbnx-build/codegen/proto_gen"
[[ -d "$CODEGEN" ]] || { echo "ERR: codegen missing — run build.sh"; exit 2; }
export PYTHONPATH="$CODEGEN:${PYTHONPATH:-}"

exec python3 -u "$PKG/scripts/atlas_register_and_launch.py"
```

### `scripts/atlas_register_and_launch.py` — 80 行的核心

```python
#!/usr/bin/env python3
import os, signal, subprocess, sys, threading, time
import grpc
import atlas_pb2 as pb
import atlas_pb2_grpc as pb_grpc

PROVIDER_ID = "my_package"          # MUST 等于 manifest 里的 name:
NAMESPACE   = "robonix/primitive/<your_ns>"   # 只是分类用，不强求是已存在的 namespace
HEARTBEAT_PERIOD_S = 30.0           # atlas 默认 90s 超时

def _log(m): print(f"[{PROVIDER_ID}] {m}", flush=True)

def main() -> int:
    pkg_root = os.environ.get(
        "RBNX_PACKAGE_ROOT",
        os.path.abspath(os.path.join(os.path.dirname(__file__), "..")))
    launch_file = os.path.join(pkg_root, "launch", "my.launch.xml")

    atlas_ep = os.environ.get("ROBONIX_ATLAS", "127.0.0.1:50051")
    stub = pb_grpc.AtlasStub(grpc.insecure_channel(atlas_ep))

    # 注意：三个注册 RPC（Primitive/Service/Skill）共享同一个 RegisterRequest message。
    try:
        stub.RegisterPrimitive(
            pb.RegisterRequest(id=PROVIDER_ID, namespace=NAMESPACE,
                               capability_md_path=""),
            timeout=5.0)
    except grpc.RpcError as e:
        _log(f"Register failed: {e.code().name} {e.details()}"); sys.exit(2)
    _log("registered")

    # 后台 heartbeat。简短 best-effort：临时网络抖动不要让整个包死。
    def _hb():
        while True:
            time.sleep(HEARTBEAT_PERIOD_S)
            try: stub.Heartbeat(pb.HeartbeatRequest(id=PROVIDER_ID), timeout=5.0)
            except grpc.RpcError: pass
    threading.Thread(target=_hb, daemon=True, name="hb").start()

    proc = subprocess.Popen(["ros2", "launch", launch_file],
                            start_new_session=True)  # 独立 PG 方便整体杀

    def _forward(sig, _):
        try: os.killpg(os.getpgid(proc.pid), signal.SIGTERM)
        except ProcessLookupError: pass
    signal.signal(signal.SIGTERM, _forward)
    signal.signal(signal.SIGINT,  _forward)

    return proc.wait()

if __name__ == "__main__":
    sys.exit(main())
```

**对应踩过的坑**：
1. `atlas_pb2.RegisterPrimitiveRequest` ❌ — 不存在。三个注册 RPC 共享 `RegisterRequest`。
2. 不发 heartbeat ❌ — 90 秒后 atlas 会把你标 TERMINATED，`rbnx caps` 看着诡异。
3. `subprocess.Popen` 不带 `start_new_session=True` ❌ — SIGTERM 时只杀父进程，`static_transform_publisher` 会变孤儿。
4. 不 `forward signal` ❌ — rbnx boot 拆除 PGID 时，Python 没机会清 ros2 launch。

---

## 3. 模板 B：典型 primitive / service

**典型场景**：你包装一个传感器（lidar、imu、camera、chassis）或一个上游 ROS 子系统（rtabmap、nav2），让它在 atlas 上注册成某个标准 contract（`robonix/primitive/lidar/lidar3d` 之类），让其它包能通过 atlas 自动发现。

**参考实现**：
- [`packages/mid360_imu_rbnx`](../packages/mid360_imu_rbnx/) — 最干净的"topic shim"模式（不开 ROS 子进程，只把已有 topic 注册到 atlas）
- [`packages/mid360_lidar_rbnx`](../packages/mid360_lidar_rbnx/) — 包装 livox launch + 注册 `/livox/cloud` 为 `primitive/lidar/lidar3d`
- [`packages/ranger_chassis_rbnx`](../packages/ranger_chassis_rbnx/) — 完整的硬件包装：CAN 初始化 + 命令收 + odom 发
- [`packages/realsense_camera_rbnx`](../packages/realsense_camera_rbnx/) — 包装 realsense2_camera launch
- [`packages/nav2_wrapper_rbnx`](../packages/nav2_wrapper_rbnx/) — service 例子（包装 nav2_bringup）

### 目录结构

```
my_primitive_rbnx/
├── package_manifest.yaml
├── README.md                    # 强烈建议，对未来调试至关重要
├── capabilities/                # 可选：包级 contract 覆盖
│   └── primitive/<ns>/...toml
├── scripts/
│   ├── build.sh
│   └── start.sh
├── my_primitive/                # python module（包名同 namespace 末段）
│   ├── __init__.py
│   └── main.py                  # 真·入口
└── src/                         # 可选：vendor 第三方 ROS 源码（如 livox_ros_driver2）
```

### `package_manifest.yaml`

```yaml
manifestVersion: 1
package:
  name: com.<vendor>.<role>.<my_primitive>
  version: 0.1.0
  vendor: <vendor>
  license: MulanPSL-2.0

build: bash scripts/build.sh
start: bash scripts/start.sh

capabilities:
  - name: robonix/primitive/<ns>/driver       # 让 rbnx boot 走 CMD_INIT/CMD_ACTIVATE
  - name: robonix/primitive/<ns>/<topic_name> # 暴露的实际数据流，例如 .../imu, .../lidar3d
```

**关键点**：声明 `*/driver` 触发 `Driver(CMD_INIT)` 握手 — rbnx boot 会把 manifest 里的 `config:` 块作为 `config_json` 通过这个 RPC 传给你。这是把部署期可调参数传进来的**唯一途径**（deploy.rs 不把 config 写文件、不注入环境变量）。

### `scripts/build.sh`（典型形态）

```bash
#!/usr/bin/env bash
set -euo pipefail
PKG="${RBNX_PACKAGE_ROOT:-$(cd "$(dirname "$0")/.." && pwd)}"
cd "$PKG"
CLEAN="${RBNX_BUILD_CLEAN:-}"
[[ "$CLEAN" == "1" ]] && rm -rf rbnx-build
mkdir -p rbnx-build/data

FLAGS=(--out-dir "$PKG/rbnx-build/codegen")
[[ "$CLEAN" == "1" ]] && FLAGS+=(--clean)
rbnx codegen -p "$PKG" "${FLAGS[@]}"

# ↓ 如果你 vendor 了 ROS 源码（src/livox_ros_driver2、src/realsense-ros 之类），
#   在这里跑 colcon build；否则跳过。
# colcon build --packages-select foo --cmake-args -DCMAKE_BUILD_TYPE=Release

touch "$PKG/rbnx-build/.rbnx-built"
```

### `scripts/start.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
PKG="${RBNX_PACKAGE_ROOT:-$(cd "$(dirname "$0")/.." && pwd)}"
cd "$PKG"

ROS_DISTRO="${ROS_DISTRO:-humble}"
set +u; source "/opt/ros/${ROS_DISTRO}/setup.bash"; set -u
# 如果有 vendor 的 ROS workspace overlay：
# [[ -f "$PKG/install/setup.bash" ]] && { set +u; source install/setup.bash; set -u; }

if ROBONIX_API="$(rbnx path robonix-api 2>/dev/null)"; then
    export PYTHONPATH="$ROBONIX_API:$PKG:${PYTHONPATH:-}"
fi
export PYTHONPATH="$PKG/rbnx-build/codegen/proto_gen:$PKG/rbnx-build/codegen/robonix_mcp_types:$PYTHONPATH"

exec python3 -m my_primitive.main
```

### `my_primitive/main.py` — 用 robonix_api（推荐）

参考 `packages/mid360_imu_rbnx/mid360_imu/main.py` 的整体结构。最简骨架：

```python
import logging, os, threading, time
from typing import Optional, Any
from robonix_api import Primitive, Ok, Err, Deferred

logging.basicConfig(level=os.environ.get("MY_LOG_LEVEL", "INFO"),
                    format="[my_primitive] %(message)s")
log = logging.getLogger("my_primitive")

# id 必须等于 deploy 的 robonix_manifest.yaml 里 `primitive: - name: ...`
my_primitive = Primitive(id="my_primitive", namespace="robonix/primitive/<ns>")

# 由 on_init 设置、由 on_activate 消费的解析后配置。
_resolved_cfg: Optional[dict[str, Any]] = None

@my_primitive.on_init
def init(cfg: dict):
    """REGISTERED → INACTIVE。
    light validation only — parse cfg, sanity check files. NO subprocess
    spawn, NO atlas-declare, NO sensor-warmup. 重活留到 on_activate。"""
    global _resolved_cfg
    cfg = cfg or {}
    try:
        timeout = float(cfg.get("sentinel_timeout_s", 30.0))
        if timeout <= 0:
            return Err(f"sentinel_timeout_s must be > 0, got {timeout}")
    except (TypeError, ValueError) as e:
        return Err(f"sentinel_timeout_s not numeric: {e}")
    _resolved_cfg = dict(cfg)
    log.info("CMD_INIT ok")
    return Ok()

@my_primitive.on_activate
def activate():
    """INACTIVE → ACTIVE。重活：spawn ROS、等第一帧、declare topic。"""
    cfg = _resolved_cfg or {}
    # 1. spawn 子进程
    # 2. sentinel: 等第一条数据，证明 pipeline 真的活了
    # 3. atlas declare
    try:
        my_primitive.declare_ros2_topic(
            "robonix/primitive/<ns>/<my_topic>",
            topic="/your/topic/name",
            qos="best_effort",
            description="你这个 topic 是干嘛的，写清楚",
        )
    except Exception as e:
        return Err(f"declare_ros2_topic failed: {e}")
    return Ok()

@my_primitive.on_deactivate
def deactivate():
    """ACTIVE → INACTIVE。kill 子进程，对称于 on_activate。"""
    return Ok()

@my_primitive.on_shutdown
def shutdown():
    """any → TERMINATED。idempotent 兜底 kill。"""
    return Ok()

if __name__ == "__main__":
    my_primitive.run()
```

### 关键设计原则（这些是吃了亏总结的）

1. **`on_init` 只做轻量校验**，重活放 `on_activate`。否则 `CMD_DEACTIVATE` → 重新 `CMD_ACTIVATE` 的路径就坏了，因为 `on_init` 的副作用没法重放。
2. **`Provider id 必须等于 deploy manifest 的 name:`**。rbnx boot 用 manifest 里的 name 做"哪个 provider 是这个 spawn 注册的"判断。不一致 = 直接 bail。
3. **`on_activate` 里要带 sentinel**：spawn ROS 进程后，等第一帧数据/服务可达再 declare topic。否则 atlas 上 capability 出现的瞬间，下游消费者就开始查询了，这时候你的 topic 可能还没起来 → 下游 init 失败。`mid360_imu/main.py::_wait_for_imu` 是个标准实现。
4. **配置只能从 `Driver(CMD_INIT)` 来**。想读 manifest 里的 `config:` 字段？只有 `on_init(cfg)` 这一个入口。
5. **失败要返回 `Err` 而不是抛异常**。lifecycle 模块抓异常但行为可能是默认 OK，会让你 silently 通过 init 然后下游莫名爆炸。
6. **可重入**：`on_deactivate`/`on_shutdown` 必须幂等。kill helper 自己 null 全局 handle，连续调两次不能报错。

### 让你的 contract 在全局 capabilities 树里有定义（必要时）

如果你声明的 contract（如 `robonix/primitive/<ns>/driver`）**不在** `<robonix>/capabilities/` 全局树里，codegen 不会为它生成 Driver Servicer，`robonix_api.lifecycle` 会跳过装载，**`@on_init` 永远不会被调用**。

两种解决方式：

- **A**: 在主 robonix 仓加一个文件 `<robonix>/capabilities/primitive/<ns>/driver.v1.toml`：
  ```toml
  [contract]
  id      = "robonix/primitive/<ns>/driver"
  version = "1"
  kind    = "primitive"
  idl     = "lifecycle/srv/Driver.srv"

  [mode]
  type = "rpc"
  ```
- **B**: 把同一个 toml 放到本包的 `capabilities/primitive/<ns>/driver.v1.toml`。`codegen.rs` 和 `deploy.rs` 都会自动把包级 capabilities 合并进全局树。

ranger 系列已有的 `primitive/{imu,lidar,camera,chassis,audio}` 都在 robonix 全局树里，直接 reuse 即可。如果你做的是新类型（譬如 `primitive/force_torque`），用方式 B 在自己包里 ship 就行，不用改主仓。

---

## 4. 模板 C：带 MCP 工具的 service / skill

**典型场景**：你想暴露给 LLM 调用的能力（导航到指定 pose、开始探索、查询场景等）。

**参考实现**：
- [`packages/explore_rbnx`](../packages/explore_rbnx/) — skill 例子（autonomous frontier exploration）
- [`packages/nav2_wrapper_rbnx`](../packages/nav2_wrapper_rbnx/) — service 例子，有完整的 navigate/status/cancel 三件套
- `robonix/system/scene/` — 复杂的 service 例子，5 个 MCP 工具

### 与模板 B 的差别

1. **`build.sh` 加 `--mcp` flag** 给 codegen，生成 MCP 类型 dataclass：
   ```bash
   FLAGS=(--out-dir "$PKG/rbnx-build/codegen" --mcp)
   ```
2. **能力声明里有 transport=mcp 的 contract**，例如 `robonix/service/navigation/navigate`。
3. **start.sh 把 `robonix_mcp_types` 加到 PYTHONPATH**：
   ```bash
   export PYTHONPATH="$PKG/rbnx-build/codegen/proto_gen:$PKG/rbnx-build/codegen/robonix_mcp_types:$PKG:${PYTHONPATH:-}"
   ```
4. **main.py 用 `Service` 而不是 `Primitive`**，并用 `@svc.mcp(contract_id)` 注册工具：
   ```python
   from robonix_api import Service, Ok, Err
   svc = Service(id="my_svc", namespace="robonix/service/<ns>")

   from <my_pkg>_mcp import MyTool_Request, MyTool_Response  # codegen 产物

   @svc.mcp("robonix/service/<ns>/my_tool")
   def my_tool(req: MyTool_Request) -> MyTool_Response:
       """这一段 docstring 直接进 LLM 的 tool description，写好它！"""
       ...
       return MyTool_Response(...)
   ```
5. **IDL 文件放在包里**：`<pkg>/capabilities/lib/<ns>/srv/MyTool.srv`（ROS srv 格式，Request/Response 字段）。codegen 会把它转成上面的 `MyTool_Request` / `MyTool_Response` dataclass。
6. **每个工具单独 declare_mcp**：参考 `system/scene/scene_service/service.py:723-740` 的 loop 写法。

### LLM 友好的 docstring 套路

LLM 通过 docstring 决定要不要、什么时候调你的工具。一些经验：
- 第一句：动词开头，描述这个工具"做"什么。
- 列出**关键输入字段**和它们的语义（坐标系、单位、可选枚举）。
- 写**返回值字段**的含义。
- 标明**错误条件**（什么情况下 `accepted=False`）。
- 用例：`packages/nav2_wrapper_rbnx/nav2_wrapper/atlas_bridge.py::navigate` 的 docstring。

---

## 5. deploy 端：把你的包接到 robonix_manifest.yaml

包做好后，要在 deploy 仓（如 `ranger_mini_deploy/`）的 `robonix_manifest.yaml` 里加一条：

```yaml
primitive:
  - name: my_primitive                        # MUST 等于 main.py 里 Primitive(id=...)
    url: https://github.com/<owner>/my_primitive_rbnx
    branch: main
    config:
      foo: bar          # 这里的内容会作为 cfg 传给 on_init(cfg)
      sentinel_timeout_s: 30.0
```

或者本地路径开发时：

```yaml
  - name: my_primitive
    path: ../packages/my_primitive_rbnx
    config: {...}
```

### 顺序很重要

`rbnx boot` 严格按 YAML 声明顺序串行启动。每个 `primitive[i]` 必须 `Driver(CMD_INIT)` 返回 `ok=true` 后，`primitive[i+1]` 才会开始。**provider 必须在 consumer 之前**，否则 consumer init 时查 atlas 找不到上游 contract，返回 `Err`，整个 boot 直接 bail（无重试）。

Ranger Mini 部署里的顺序是：

```
ranger_description (静态 TF)
  → mid360_lidar (lidar 也产生 /livox/imu 副作用)
  → mid360_imu (订阅 /livox/imu)
  → realsense_camera
  → ranger_chassis (产生 /odom)

service: mapping (rtabmap, 消费 lidar3d + rgb + depth + odom)
       → nav2 (消费 mapping/occupancy_grid)

skill:  explore (lazy-activate)
```

### 顶层 `env:` 块会被 propagate

deploy.rs 第 531 行：

```rust
for (k, v) in &deploy.env {
    unsafe { std::env::set_var(k, expand_env_in_str(v)); }
}
```

manifest 顶层 `env:` 会注入到 `rbnx boot` 进程自己的环境，然后被所有子包继承。这是把环境变量传到包里的**唯一**机制（manifest 里的 `config:` 是 RPC 传，不进环境）。我们用它做 docker/native 模式开关、CN 网络下的镜像 host 配置等。

---

## 6. 工具链速查

```bash
# 在新仓里 scaffold 一个空模板
rbnx package new --name my_primitive_rbnx --kind primitive

# 单包构建
cd my_primitive_rbnx && bash scripts/build.sh
# 或：
rbnx build -p my_primitive_rbnx

# 部署整个 manifest
rbnx boot

# 干净重来（清掉 cache + build）
rbnx clean my_primitive   # 单包
rbnx clean                 # 全部

# 查看 atlas 当前注册了什么
rbnx caps -v               # provider + 所有 capabilities
rbnx contracts             # 全局 contract 注册表
rbnx tools                 # MCP 工具列表（LLM 看到的就是这个）

# Cache 在哪
ls rbnx-boot/cache/<package_name>/

# 包的运行时产物
ls rbnx-boot/cache/<package_name>/rbnx-build/data/<my>.log
```

---

## 7. 常见坑（按调试时长排序）

### 坑 1: `no robonix/<ns>/driver contract — skipping lifecycle servicer` ⚠️

**症状**：rbnx boot 看到 `package transitions REGISTERED → ACTIVE` 但 `Driver(CMD_INIT)` 没被调用、`@on_init` 永远没跑，下游因为缺数据全部 fail。

**根因**：声明的 `<ns>/driver` contract 没有 toml 定义在全局或包级 `capabilities/` 里。codegen 不为它生成 Servicer，`robonix_api.lifecycle.build_lifecycle_servicer` 跳过装载。

**修法**：参考第 3 节末尾"让你的 contract 在全局 capabilities 树里有定义"。

### 坑 2: rbnx boot 卡在 `wait_for_registration`

**症状**：`[ - ] my_pkg  registering with atlas… 90.0s` 然后 timeout 报错。

**根因**：你的 start 进程根本没向 atlas 调过 `RegisterPrimitive/Service/Skill`。最常见就是 start.sh 直接 `exec ros2 launch`、`exec foo_node` 没用 robonix_api。

**修法**：用模板 A 的 `atlas_register_and_launch.py`，或者用 `robonix_api.Capability` 的标准生命周期 — 它的 `bootstrap()` 会自动注册。

### 坑 3: `provider_id mismatch`

**症状**：`[ ✗ ] my_pkg  provider_id mismatch: manifest says name='my_pkg' but Capability(id='foo') registered`

**根因**：deploy manifest 里 `name: my_pkg` 但 main.py 里 `Primitive(id="foo")`。rbnx boot 用前者匹配后者，不一致就 bail。

**修法**：让两边一致。建议 main.py 用 `id=os.environ.get("ROBONIX_CAPABILITY_ID", "<default>")`，部署侧再覆盖。

### 坑 4: package ACTIVE 后 `rbnx caps` 突然变 TERMINATED

**根因**：没发 heartbeat。atlas 默认 90 秒超时（`DEFAULT_HEARTBEAT_TIMEOUT_MS`）。

**修法**：用 robonix_api 时 `Capability.bootstrap()` 自动起后台心跳，不用管。如果手撸 atlas RPC，必须自己起心跳线程（30 秒一次比较安全）。

### 坑 5: 子进程不被 SIGTERM 杀干净

**症状**：rbnx boot 退出后，`ps aux | grep ros2` 还有一堆 orphan。下次 boot port 被占用各种诡异。

**根因**：spawn 子进程没用 `start_new_session=True`，或没 forward signal。

**修法**：`Popen(..., start_new_session=True)` + `os.killpg(os.getpgid(child.pid), SIGTERM)`。

### 坑 6: `RegisterPrimitiveRequest` 不存在

**症状**：`AttributeError: module 'atlas_pb2' has no attribute 'RegisterPrimitiveRequest'`

**根因**：以为按 RPC 名字会有专门 message。实际上三个注册 RPC（Primitive/Service/Skill）共享 `RegisterRequest`。

**修法**：用 `pb.RegisterRequest(id=..., namespace=..., capability_md_path=...)`。

### 坑 7: cache 旧、push 没生效

**症状**：本地修了 package 代码、push 到 GitHub，重启 rbnx boot，行为没变化。

**根因**：`rbnx boot` 看到 `rbnx-boot/cache/<name>/` 已存在就**不会 pull**，跳过 clone。改 cache 才是有效路径。

**修法**：

```bash
rbnx clean my_pkg     # 或者直接 rm -rf rbnx-boot/cache/my_pkg
rbnx boot
```

### 坑 8: docker mode 和 native mode 的选择

`system/scene` 用了 docker/native 双路径模式（参考 `mapping_rbnx`），通过环境变量 `ROBONIX_<UPPER>_FORCE=native|docker` 或 `ROBONIX_<UPPER>_PLATFORM=jetson_orin` 切换。如果你的包要跑在 Jetson 单机部署上、不想要 docker overhead，参考这两个包的 `scripts/start.sh` 做双路径。

---

## 8. 完整工作流（new package from scratch）

```bash
# 1. 需求分析：你是 primitive / service / skill？要不要 MCP 工具？
#    → 选模板 A / B / C

# 2. scaffold
rbnx package new --name my_primitive_rbnx --kind primitive
cd my_primitive_rbnx

# 3. 改 package_manifest.yaml：填 name / capabilities

# 4. 写 main.py（or 模板 A 的 atlas_register_and_launch.py）

# 5. 如果 capability namespace 不在全局树，加 capabilities/<...>/<driver>.v1.toml

# 6. 本地试跑：
bash scripts/build.sh
# 单独跑（需要先有 atlas 进程）：
ROBONIX_ATLAS=127.0.0.1:50051 bash scripts/start.sh

# 7. 加到 deploy manifest，rbnx boot 端到端

# 8. 看 rbnx caps -v / rbnx contracts / rbnx tools 验证
```

---

## 9. 推荐的代码组织

读 `packages/mid360_imu_rbnx/` 是 ~30 分钟入门，强烈建议。目录就 4 个文件：

```
mid360_imu_rbnx/
├── README.md                    ← 必读，写清楚 contract 表 + boot ordering 要求
├── package_manifest.yaml        ← 12 行
├── scripts/
│   ├── build.sh                 ← 标准 codegen
│   └── start.sh                 ← 标准 ROS source + python -m
└── mid360_imu/
    ├── __init__.py
    └── main.py                  ← 150 行实现完整 lifecycle + sentinel
```

最大的两个观感原则：
- **README 必写**。把"这个包是干嘛的"、"消费哪些 contract"、"暴露哪些 contract"、"必须在哪个包之后启动"这四件事写在 README 第一屏。Ranger Mini bring-up 时调试时间 70% 花在没读 README 上。
- **每个非显然的设计决定写注释**。`mid360_imu/main.py` 的 `_wait_for_imu` 注释解释了为什么用 best_effort QoS、为什么 spin 完就 destroy。这种注释对 6 个月后的自己价值连城。

---

## 10. 进一步阅读

- `/Users/howenliu/lab/robonix/CLAUDE.md` — 框架内部生命周期 + atlas 状态机的精确定义
- `/Users/howenliu/lab/robonix/system/scene/README.md` — 系统服务最复杂的例子，覆盖 docker/native 双路径
- `/Users/howenliu/lab/ranger_mini_deploy/HANDOFF.md` — Ranger 部署上的运维知识
- `/Users/howenliu/lab/ranger_mini_deploy/robonix_manifest.yaml` — 一个真实的、注释详尽的 manifest 范本
- `/Users/howenliu/lab/robonix/rust/crates/robonix-cli/src/cmd/deploy.rs` — `spawn_and_init` / `wait_for_registration` 的源码注释，比官方文档详细

---

文档生成时基于：
- `ranger_mini_deploy/robonix_manifest.yaml`（含 ranger_description / mid360_lidar / mid360_imu / realsense_camera / ranger_chassis / mapping / nav2 / explore 8 个 ranger 系列包）
- `packages/` 下 8 个 `*_rbnx` 包的实际代码
- 本次部署中真实踩过的 8 个坑（章节 7）
