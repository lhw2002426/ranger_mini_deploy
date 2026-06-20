# Soma 接入部署 cheatsheet

> 给你（在部署机上的人）看的速查页。完整设计 / 决策依据见
> `/Users/howenliu/lab/docs/ranger_mini_soma_integration_plan.md` §A。

## 0. 触发条件

执行下面任何一步之前确认：

- [ ] PR91 已经合到 `syswonder/robonix:main`（或显式选择走 PR 分支兜底，见 §A.1）
- [ ] 当前线上栈正常（避免接入 Soma 时撞坏在跑的任务）

## 1. 本机已经为你准备好的文件

```
ranger_mini_deploy/
├── soma.ymal                                ← 新增（本机产出）
├── soma_config.local.yaml                   ← 新增（本机产出）
├── urdf/
│   └── ranger_mini.urdf                     ← 新增（本机产出，含底盘+雷达+相机 mount）
├── robonix_manifest.yaml                    ← 不动！按下面的 patch 文件指引手动编辑
├── robonix_manifest.yaml.soma.patch         ← 新增（本机产出，是说明文档不是 diff）
└── SOMA_DEPLOY.md                           ← 新增（你正在读）
```

## 2. 在部署机上要做的事（按顺序）

### 2.1 编 robonix-soma binary

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
scp soma.ymal                              "$DEPLOY_REMOTE"
scp soma_config.local.yaml                 "$DEPLOY_REMOTE"
scp urdf/ranger_mini.urdf                  "$DEPLOY_REMOTE/urdf/"
scp robonix_manifest.yaml.soma.patch       "$DEPLOY_REMOTE"   # 留作参考；如不需要可省
scp SOMA_DEPLOY.md                         "$DEPLOY_REMOTE"
```

> 部署机上 `urdf/` 子目录已存在（README 占位用），scp 直接落进去即可。

### 2.3 静态校验

```bash
ssh robot
cd ~/lhw/ranger_mini_deploy/
rbnx validate
```

如果 `rbnx validate` 报"unknown key system.soma"，说明部署机上 `rbnx` 还不认这个 key（PR91 dispatcher 没合主）→ 切到方案 2：**回滚** manifest（`mv robonix_manifest.yaml.before-soma robonix_manifest.yaml`），再跳到 §2.5。

### 2.4 启动 + 验收（方案 1：rbnx boot 自动拉起 soma）

```bash
ssh robot
cd ~/lhw/ranger_mini_deploy/
rbnx boot
```

另开 ssh 跑验收命令：

```bash
# 9 条验收
ls soma.ymal urdf/ranger_mini.urdf soma_config.local.yaml
robonix-soma --help                          # 已 install
rbnx caps | grep soma                        # 两条 get_ymal / get_urdf 都 ACTIVE
rbnx caps -v | grep -A 3 'robonix/system/soma'
                                             # endpoint = 127.0.0.1:50091, transport = grpc

# gRPC 直连（用 grpcurl）
grpcurl -plaintext -d '{"robot_id":""}' 127.0.0.1:50091 \
    robonix.contracts.RobonixSystemSomaGetYmal/GetYmal
# 期望：返回 ymal_text 包含 "Ranger Mini"
grpcurl -plaintext -d '{"robot_id":""}' 127.0.0.1:50091 \
    robonix.contracts.RobonixSystemSomaGetUrdf/GetUrdf
# 期望：返回 urdf_xml 包含 <robot name="ranger_mini_v2"> 和 <link name="livox_frame"/>

# 整套栈不退化（mapping/nav2/explore 仍正常）
rbnx caps | grep -v ACTIVE      # 期望：除了 explore (INACTIVE) 之外没东西
```

### 2.5 兜底（方案 2：rbnx boot 还不认 system.soma）

```bash
# 终端 1
ssh robot
robonix-atlas

# 终端 2
ssh robot
cd ~/lhw/ranger_mini_deploy/
robonix-soma --config ./soma_config.local.yaml

# 终端 3
ssh robot
cd ~/lhw/ranger_mini_deploy/
rbnx boot
```

后续验收同 §2.4。

## 3. 出错时

| 症状 | 原因 | 处理 |
|---|---|---|
| `robonix-soma: command not found` | install 失败 | 回 §2.1 重做 `make install` |
| `rbnx caps` 看不到 soma 两条 cap | atlas 没起，或 soma 启动后 crash | `rbnx logs` / `journalctl` 查 soma 进程 stdout |
| `unknown Soma robot_id ''` | soma_config.local.yaml 的 `default_robot` 拼错或 `soma.ymal` 的 `robot.id` 不一致 | 两处都应是 `ranger_mini_01` |
| `no Soma YMAL/YAML file found` | soma 进程 cwd 不对，没找到 deployment dir | 用 `--config` 传**绝对路径**或者从 deploy 目录里跑 soma |
| `<robot name="ranger_mini_v2">` 但缺 `<link name="livox_frame"/>` | URDF 没拷全 / 拷的是旧版 | 重新 scp `urdf/ranger_mini.urdf` |
| 整套栈起来后 rtabmap 报 TF 失败 | URDF 里的 placeholder 数字与现实差太多，但更可能是 ranger_description 没起来——本阶段 ranger_description 仍是 TF 主源 | 检查 `rbnx caps` 里 `ranger_description` 是 ACTIVE |

## 4. 完成后

回到本机 `/Users/howenliu/lab/docs/ranger_mini_soma_integration_plan.md` §0，把状态从 v0.2 → v0.3，并在 §A.1 / §A.4 末尾加 `[done @YYYY-MM-DD]`。

阶段 B/C 的部署侧动作分别参照那份文档的 §A.6 / §A.7。
