# Soma 接入部署 cheatsheet

> 给你（在部署机上的人）看的速查页。完整设计 / 决策依据见
> `/Users/howenliu/lab/docs/ranger_mini_soma_integration_plan.md` §A。
> PR91 校核结论与本文件对齐说明见
> `/Users/howenliu/lab/docs/ranger_mini_deploy_pr91_soma_check.md`。

## 0. 触发条件

执行下面任何一步之前确认：

- [x] PR91 已合入部署机上的 robonix 源码（即 `system/soma/`、`capabilities/system/soma/`、`tools/rbnx/.../deploy.rs` 里的 soma builtin 分支都已就位）
- [ ] 当前线上栈正常（避免接入 Soma 时撞坏在跑的任务）

## 0.5 启动机制速览（PR91 源码事实）

> 这一节是新加的：让接手的人**别再去删 `soma_bridge` 这个 skill**。

```
rbnx boot
  ├─ system builtin 阶段（rbnx 直接 fork，按 bin_map 顺序）
  │   atlas → executor → pilot → liaison → soma
  │     ↑
  │     bin_map 见 tools/rbnx/src/cmd/deploy.rs:593-599
  │     soma 在 PR91 已加入；CLI flag 翻译规则在 :1156-1178
  │
  ├─ 非 builtin system 阶段（走 rbnx start -p + Driver(CMD_INIT)）
  │   memory / scene
  │
  ├─ primitive / service 阶段（走 rbnx start -p + Driver(CMD_INIT)）
  │   ranger_description → mid360_lidar → mid360_imu → realsense_camera
  │   → ranger_chassis → mapping → nav2
  │
  └─ skill 阶段（lazy-activate）
      explore / soma_bridge      ← INACTIVE 直到 pilot 第一次 MCP 调用
```

`rbnx boot` 把 `system.soma:` 翻译成 `robonix-soma --listen ... --atlas
... --config ... --provider-id ... --default-robot ... --log ...` 直接
fork（`spawn_system_binary` 走 `Command::new(bin)`，**不会** chdir，
所以 `--config` 用绝对路径最稳）。

**soma 起来后做两件事**：

1. 自己向 atlas 注册两条 cap（`system/soma/src/main.rs:62-92`）：
   ```
   robonix/system/soma/get_yaml   Transport::Grpc, port 50091
   robonix/system/soma/get_urdf   Transport::Grpc, port 50091
   ```
2. 起 `PackageSupervisor` 调 `PackageLauncher` 二次 fork
   `rbnx start -p <deployment-pkg>` —— 但我们在
   `soma_config.local.yaml` 里把 `start_packages: false` 关掉了，
   所以这一步**实际不 fork**，包仍由 rbnx boot 自己起，避免与
   soma 抢着 spawn。

### 为什么还要保留 `skill.soma_bridge`

pilot 的 LLM 工具发现循环（`system/pilot/src/discovery.rs:74-90`）
写死了 `query_capabilities("", "", Transport::Mcp)`，对
`cap.transport != Mcp` 的 cap **直接 `continue` 丢弃**。也就是说：

* `rbnx caps | grep soma` 能看见两条 grpc cap（soma 注册成功）。
* 但 pilot 的 LLM 永远看不到，因此无法 "去问 soma get_yaml"。

`soma_bridge_rbnx`（this repo 的 `skill:` 段最后一项）是一个轻量
fastmcp server，把那两条 cap 用 `Transport::Mcp` 重新登记一份，handler
内部以 gRPC 客户端身份转发到真 soma。**删它 = LLM 直接断联**。

可以拆除的两个上游触发条件：

* (a) pilot 的 discovery 改成读所有 transport（带 grpc→tool 适配），或
* (b) pilot 直接把 `soma.yaml` 渲进 system prompt（计划里说过的方案）。

任一种合并后，把 manifest 里 `- name: soma_bridge` 整段删掉即可。

## 1. 本机已经为你准备好的文件

```
ranger_mini_deploy/
├── soma.yaml                                ← 新增（本机产出）
├── soma_config.local.yaml                   ← 新增（本机产出）
├── urdf/
│   └── ranger_mini.urdf                     ← 新增（本机产出，含底盘+雷达+相机 mount）
├── robonix_manifest.yaml                    ← 已加 system.soma 块 + soma_bridge skill
└── SOMA_DEPLOY.md                           ← 新增（你正在读）
```

## 2. 在部署机上要做的事（按顺序）

### 2.1 编 robonix-soma binary（整套同源 install）

```bash
ssh robot
export ROBONIX_SOURCE_PATH=/home/syswonder/wheatfox/robonix   # 或 cyt 那份，二选一
cd "$ROBONIX_SOURCE_PATH"

git fetch origin
git checkout main
git pull --ff-only

make build && make install   # 整套同源 install，避免 atlas/soma 半新半旧

which robonix-soma && robonix-soma --help
```

> ⚠️ 不要单独 `cargo install -p robonix-soma`。要么整套 install，要么不动。原因见
> `docs/ranger_mini_soma_integration_plan.md` §3.1.1。

### 2.2 把本机产出的全部文件 scp 到部署机（一次覆盖）

本机的 `robonix_manifest.yaml` 已经把 `system.soma:` 块加好，scp 过去**直接覆盖**部署机的同名文件即可，无需再手编。

> ⚠️ 覆盖前**强烈建议**先在部署机上把现有 manifest 备份一份：
> `ssh robot 'cp ~/lhw/ranger_mini_deploy/robonix_manifest.yaml ~/lhw/ranger_mini_deploy/robonix_manifest.yaml.before-soma'`

在你的 Mac 上：

```bash
cd /Users/howenliu/lab/ranger_mini_deploy
DEPLOY_REMOTE=robot:~/lhw/ranger_mini_deploy/

scp robonix_manifest.yaml                  "$DEPLOY_REMOTE"   # ★ 覆盖
scp soma.yaml                              "$DEPLOY_REMOTE"
scp soma_config.local.yaml                 "$DEPLOY_REMOTE"
scp urdf/ranger_mini.urdf                  "$DEPLOY_REMOTE/urdf/"
scp SOMA_DEPLOY.md                         "$DEPLOY_REMOTE"
```

> 部署机上 `urdf/` 子目录已存在（README 占位用），scp 直接落进去即可。

### 2.3 静态校验

```bash
ssh robot
cd ~/lhw/ranger_mini_deploy/
rbnx validate
```

如果 `rbnx validate` 报"unknown key system.soma"，说明部署机上 `rbnx` **不是 PR91**（PR91 已把 soma 列入 SYSTEM_BUILTINS，`tools/rbnx/src/cmd/deploy.rs:598` + `:691` + `clean.rs:157` 三处一致）→ 回到 §2.1 重做 `make install`，必要时 `git log -- tools/rbnx/src/cmd/deploy.rs | head` 确认源码版本。

### 2.4 启动 + 验收

```bash
ssh robot
cd ~/lhw/ranger_mini_deploy/
rbnx boot
```

PR91 的 rbnx boot 会把 `system.soma:` 翻译成

```
robonix-soma \
  --listen 127.0.0.1:50091 \
  --atlas 127.0.0.1:50051 \
  --config /home/syswonder/lhw/ranger_mini_deploy/soma_config.local.yaml \
  --provider-id soma \
  --default-robot ranger_mini_01 \
  --log info
```

并直接 fork，无需另开终端。翻译规则在
`tools/rbnx/src/cmd/deploy.rs:1156-1178` 的 `"soma" =>` 分支；启动方式
是 `spawn_system_binary` 走的 `Command::new(bin)`（line 328-340），不
经过 `Driver(CMD_INIT, config_json)`，所以 manifest 里 `system.soma:`
下的每个字段都必须对应一个支持的 CLI flag。

另开 ssh 跑验收命令：

```bash
# 验收
ls soma.yaml urdf/ranger_mini.urdf soma_config.local.yaml
robonix-soma --help                          # 已 install

# soma 自己注册的两条 grpc cap
rbnx caps | grep soma                        # 期望：soma + soma_bridge 都看得到
rbnx caps -v | grep -A 3 'robonix/system/soma'
                                             # endpoint = 127.0.0.1:50091
                                             # transport = grpc  ← 注意是 grpc 不是 mcp
                                             # 这就是为何还需要 soma_bridge：
                                             # pilot 的 LLM 发现循环只读 mcp。

# gRPC 直连（用 grpcurl）—— 验证 soma 本身工作
grpcurl -plaintext -d '{"robot_id":""}' 127.0.0.1:50091 \
    robonix.contracts.RobonixSystemSomaGetYaml/GetYaml
# 期望：返回 yaml_text 包含 "Ranger Mini"
grpcurl -plaintext -d '{"robot_id":""}' 127.0.0.1:50091 \
    robonix.contracts.RobonixSystemSomaGetUrdf/GetUrdf
# 期望：返回 urdf_xml 包含 <robot name="ranger_mini_v2"> 和 <link name="livox_frame"/>

# soma_bridge 两条 mcp cap（lazy-activate，先看到 INACTIVE 正常）
rbnx caps -v | grep -A 3 'robonix/skill/soma_bridge'
                                             # 期望：两条 transport=mcp
                                             # 状态 INACTIVE → 等 pilot 第一次调用

# 端到端走 pilot LLM（设计意图的真实验证）
rbnx ask "去问 soma_bridge get_yaml，你的 description 段告诉我"
# pilot 应该：选 get_yaml 这个 MCP 工具 → 调用 → 拿到 yaml_text →
# 总结 description.summary / can_do / cannot_do 给你。

# 整套栈不退化（mapping/nav2/explore 仍正常）
rbnx caps | grep -v ACTIVE      # 期望：除了 explore + soma_bridge (INACTIVE) 之外没东西
```

> 注：`soma_bridge` 是 lazy-activate skill — `rbnx caps` 第一次会显示 INACTIVE，直到 pilot
> 第一次通过 MCP 调它的 `get_yaml` / `get_urdf`，executor 才发 `CMD_ACTIVATE`。
> 想提前预热可手动 `rbnx skill activate soma_bridge`（或让 pilot 跑一次 list_tools）。
>
> ⚠️ **不要因为 `rbnx caps` 里 `robonix/system/soma/*` 显示 ACTIVE 就觉得
> bridge 是冗余的而想删它**。看上面 §0.5 "为什么还要保留 `skill.soma_bridge`"
> ——soma 注册的是 grpc cap，pilot 的 LLM 工具发现写死了只读 mcp，所以
> 在 pilot 上游修好之前，bridge 是 LLM ↔ soma 的**唯一通路**。

### 2.5 手工启动 soma（仅用于调试/排错）

正常 `rbnx boot` 已经覆盖 soma 启动，**无需** 另开终端跑 `robonix-soma`。
仅当你要单独定位 soma 启动失败的根因时才这么做：

```bash
ssh robot
robonix-soma --config /home/syswonder/lhw/ranger_mini_deploy/soma_config.local.yaml \
             --listen 127.0.0.1:50091 \
             --atlas 127.0.0.1:50051
```

此时需要先把 manifest 里 `system.soma:` 整段注释掉再 `rbnx boot`，否则 rbnx 会再拉一份 soma 起来抢端口 50091。

## 3. 出错时

| 症状 | 原因 | 处理 |
|---|---|---|
| `robonix-soma: command not found` | install 失败 / PATH 没刷新 | 回 §2.1 重做 `make install`，或 `source ~/.cargo/env` |
| `rbnx validate` 报 `unknown key system.soma` | rbnx 不是 PR91 | 回 §2.1 重做 `make install` |
| `rbnx caps` 看不到 soma 两条 cap | atlas 没起，或 soma 启动后 crash | `rbnx logs soma`、或 `journalctl --user -u rbnx*` 查 soma 进程 stdout |
| `unknown Soma robot_id ''` | `soma_config.local.yaml` 的 `default_robot` 拼错或 `soma.yaml` 的 `robot.id` 不一致 | 两处都应是 `ranger_mini_01` |
| `no Soma YAML file found in ''` | soma 配置里 `robonix_root` 或 `deployments` 解析为空 PathBuf（理论上 PR91 已修复，仍出现说明 yaml 写法异常） | 用 `--config` 传**绝对路径**，并把 `robonix_root` / `deployments` 都写绝对路径（见本仓 `soma_config.local.yaml`） |
| `<robot name="ranger_mini_v2">` 但缺 `<link name="livox_frame"/>` | URDF 没拷全 / 拷的是旧版 | 重新 scp `urdf/ranger_mini.urdf` |
| `grpcurl` 第一次调 `soma_bridge` 的 MCP cap 报 `cap inactive` | lazy-activate 第一次必然 miss | 再调一次即可；或预热 `rbnx skill activate soma_bridge` |
| 整套栈起来后 rtabmap 报 TF 失败 | URDF 里的 placeholder 数字与现实差太多，但更可能是 ranger_description 没起来——本阶段 ranger_description 仍是 TF 主源 | 检查 `rbnx caps` 里 `ranger_description` 是 ACTIVE |

## 4. 完成后

回到本机 `/Users/howenliu/lab/docs/ranger_mini_soma_integration_plan.md` §0，把状态从 v0.2 → v0.3，并在 §A.1 / §A.4 末尾加 `[done @YYYY-MM-DD]`。

阶段 B/C 的部署侧动作分别参照那份文档的 §A.6 / §A.7。
