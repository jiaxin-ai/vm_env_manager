# VM Manager v1

## 1. 目标与边界

- 实现 `api.md` 中的 6 个接口。路径、字段、返回码都以 `api.md` 为准，不得增删或改名。实现中如果发现接口规范有问题，记录到 TEST_REPORT.md 并报告，不要自行修改 `api.md`。
- 每台虚拟机对应一个 Docker 容器。以下是已验证可用的启动命令，复用它，只做 7.4、7.5 规定的修改，不研究镜像内部的启动机制：

```bash
docker run -it --rm \
    -p 8006:8006 \
    -p 3390:3389 \
    -p 5000:5000 \
    --name winarena \
    --platform linux/amd64 \
    --device=/dev/kvm \
    -e RAM_SIZE=8G -e CPU_CORES=4 \
    -v /data/hdd/ProSoftAgentArena/src/win-arena-container/vm/storage/.:/base_storage:ro \
    --privileged \
    --cap-add NET_ADMIN \
    --stop-timeout 120 \
    --entrypoint /bin/bash julyai/mywinarena:v1 \
    -c 'echo '"'"'1. Construct basic env...'"'"'; \
    rm -rf /storage; \
    mkdir -p /storage; \
    cp -n /base_storage/windows* /storage/; \
    rm -f /storage/windows*.vars; \
    qemu-img create -f qcow2 -F raw -b /base_storage/data.img /storage/data.qcow2; \
    echo '"'"'2. Starting VM...'"'"'; \
    nohup /usr/bin/tini -s /run/entry.sh > /dev/null 2>&1 & sleep 1; \
    echo '"'"'3. Applying iptables bypass rules...'"'"'; \
    iptables -t nat -I PREROUTING 1 -p tcp --dport 8000 -j RETURN; \
    iptables -t nat -I PREROUTING 1 -p tcp --dport 8080 -j RETURN; \
    iptables -t nat -I PREROUTING 1 -p tcp --dport 22 -j RETURN; \
    tail -f /dev/null'
```

- 单台 Linux 宿主机，服务直接运行在宿主机上：单进程，单个 uvicorn worker，不使用 `--reload`。
- 资源不足时排队，严格先进先出（FIFO）。支持排队超时和运行到期自动停止。服务重启后，状态和队列都不丢失。
- 只使用预装软件的基础磁盘，不自动安装软件。
- **v1 不做**：本文档未定义的功能，如鉴权、快照、GPU、多宿主机等。
- **代码要简单通用、人类易读、模块职责清楚、方便后续迭代。能简单就简单：不做本文没要求的抽象、优化和兜底，不引入插件框架或通用平台。**

## 2. 技术栈

| 用途 | 选型 |
| --- | --- |
| 语言 | Python 3.10+ |
| Web 框架 | FastAPI + Pydantic v2 + uvicorn。Docker SDK 是同步调用，路由用普通 `def` |
| Docker 操作 | `docker`（Docker SDK for Python） |
| 数据库 | 标准库 `sqlite3`，开启 WAL |
| HTTP 探测 | `requests`，设置 `trust_env=False`，不继承环境代理 |
| 测试 | `pytest` |

## 3. 目录结构

```
vm-manager/
├── api.md                   # 接口规范（已提供，不要修改）
├── task.md                  # 本文档，定义开发任务与验收标准
├── main.py                  # 读取配置、启动前检查、恢复状态、启动巡检线程、启动 uvicorn
├── api.py                   # 6 个接口、请求/响应模型、错误转换；不直接操作 Docker 或 SQL
├── manager.py               # 校验与环境匹配、资源计算、排队调度、巡检、停止/删除、实例锁
├── docker_backend.py        # Docker SDK 调用、端口读取、健康检查
├── db.py                    # SQLite 建表与读写函数
├── profiles.json            # 环境配置（见 4.1）
├── settings.json            # 服务参数（见 4.2）
├── scripts/start_windows.sh # 容器启动脚本（见 7.5）
├── tests/
├── requirements.txt
├── README.md                # 安装、配置、启动、curl 示例、常见问题
└── TEST_REPORT.md           # 真实测试的条件、请求、结果和未完成项
```

`docker_backend.py` 对外只暴露少量函数（如 `create_and_start`、`list_managed`、`check_health`、`stop`、`remove`），方便测试时替换为假实现。

## 4. 配置

配置在服务启动时读取，修改后需重启才生效。每台虚拟机创建时保存一份配置副本，之后修改配置不影响已有的虚拟机。相对路径一律按项目根目录解析。

### 4.1 profiles.json

```json
[
  {
    "id": "win11-autocad2026",
    "os": "windows11",
    "software": ["autocad-2026"],
    "docker_image": "julyai/mywinarena:v1",
    "base_storage": "/data/hdd/ProSoftAgentArena/src/win-arena-container/vm/storage",
    "base_img": "data_v1_autocad_30G.img",
    "base_format": "raw",
    "probe_path": "/probe",
    "min_cpu_cores": 4,
    "min_ram_gb": 8
  },
  {
    "id": "win11-autocad2026-solidworks2025-ansys2024r1",
    "os": "windows11",
    "software": ["autocad-2026", "solidworks-2025", "ansys-2024-r1"],
    "docker_image": "julyai/mywinarena:v1",
    "base_storage": "/data/hdd/ProSoftAgentArena/src/win-arena-container/vm/storage",
    "base_img": "data.img",
    "base_format": "raw",
    "probe_path": "/probe",
    "min_cpu_cores": 4,
    "min_ram_gb": 8
  }
]
```

- **匹配规则**：`os` 精确匹配；`software` 按集合精确匹配，忽略顺序。
- 文件名中的 `30G` 不作为任何判断依据。
- 容器内部端口固定为 VNC 8006、RDP 3389、Agent 5000，在代码中写成常量。

### 4.2 settings.json

```json
{
  "listen_host": "0.0.0.0",
  "listen_port": 8000,
  "access_host": "REPLACE_WITH_HOST_LAN_IP",
  "data_dir": "/data/hdd/vm-manager-data",
  "host_reserved_cpu": 2,
  "host_reserved_ram_gb": 4,
  "vm_overhead_ram_gb": 1,
  "min_free_disk_gb": 50,
  "default_timeout_minutes": 120,
  "default_queue_timeout_minutes": 30,
  "boot_timeout_seconds": 900,
  "poll_interval_seconds": 2,
  "probe_timeout_seconds": 3,
  "health_failure_threshold": 3,
  "stop_timeout_seconds": 120
}
```

| 字段 | 说明 |
| --- | --- |
| `access_host` | 接口返回的 `access.host`，必须是调用方能访问到的宿主机 IP 或域名 |
| `data_dir` | 存放数据库文件 `vmm.db` |
| `min_free_disk_gb` | Docker 数据目录所在磁盘至少要保留的空间。虚拟机磁盘存放在容器中，会占用这块磁盘 |

所有 `_gb` 字段都按 GiB（1024³ 字节）计算。

### 4.3 启动前检查

以下任一项不满足，服务都启动失败，并给出明确的错误信息：

1. profile 的 `id` 不能重复，「os + 软件集合」也不能重复。`base_img` 只能是文件名，不能包含 `/` 或 `..`。
2. `access_host` 不是占位值。
3. `/dev/kvm` 存在，Docker 可以连接，镜像已在本地。服务运行期间不拉取镜像。
4. 每个 profile 的 `base_img` 文件都存在。
5. 宿主机上有 `qemu-img` 时，用 `qemu-img info --output=json` 只读检查 `base_img` 的实际格式，必须与 `base_format` 一致；没有 `qemu-img` 时跳过此项，并记录一条 warning。

## 5. 数据库

只用一张表 `vms`。删除虚拟机时，整行删除。

| 字段 | 说明 |
| --- | --- |
| seq INTEGER PK AUTOINCREMENT | 排队顺序以此为准 |
| vm_id TEXT UNIQUE | 8 位小写十六进制 |
| name、os、software、cpu_cores、ram_gb、expose、timeout_minutes、queue_timeout_minutes | 请求内容；`software`、`expose` 存为 JSON 文本 |
| profile_json | 匹配到的 profile 的完整副本 |
| status | 8 种状态之一 |
| container_id、ports | 容器 ID，以及宿主机端口（JSON 文本：`{"vnc":..,"rdp":..,"agent":..}`） |
| reserved INTEGER | 1 表示正在占用 CPU/内存。从 queued 变为 creating 时置 1，**确认容器已停止后**才置 0 |
| health_failures INTEGER | running 状态下连续检查失败的次数 |
| queue_deadline、queue_reason | 排队截止时间（`created_at + queue_timeout_minutes`）；队首最近一次的等待原因 |
| created_at、started_at、expires_at、stopped_at | UTC ISO 8601，精确到秒。`started_at` 是离开队列（变为 creating）的时间 |
| stop_reason | manual / cancelled / expired / failure；从未运行过的失败虚拟机为 null |
| error_code、error_message | 异步失败的原因；完整异常同时写入日志 |

- **所有状态变更都用条件更新（CAS）**：`UPDATE ... WHERE vm_id=? AND status IN (...)`，并检查受影响的行数。这样在取消与领取、删除与创建同时发生时，只会有一方成功。
- 每个线程使用独立的数据库连接，事务要短。**不要在持有事务时等待 Docker 或网络。**

## 6. 状态与资源

### 6.1 状态流转

| 从 | 到 | 触发条件 | 执行方 |
| --- | --- | --- | --- |
| （新建） | queued | 创建请求通过全部校验 | 创建接口 |
| queued | creating | 轮到队首且通过准入检查（6.4）；同时预留资源，写入 `started_at` 和 `expires_at` | 巡检线程 |
| queued | failed | 超过 `queue_deadline`（QUEUE_TIMEOUT），或前提条件失效（HOST_NOT_READY） | 巡检线程 |
| queued | stopped | 排队中被取消（cancelled） | 停止接口 |
| queued | （删除记录） | 删除请求 | 删除接口 |
| creating | starting | 容器启动成功，端口已读出 | 巡检线程 |
| creating | failed | 创建或启动容器失败（PROVISION_FAILED）；清理残留容器，释放资源 | 巡检线程 |
| starting | running | 健康检查通过（7.6） | 巡检线程 |
| starting | failed | 超过 `started_at + boot_timeout_seconds`（STARTUP_TIMEOUT），或容器已退出（CONTAINER_EXITED） | 巡检线程 |
| running | failed | 容器已退出（CONTAINER_EXITED），或连续检查失败达到阈值（HEALTH_CHECK_FAILED） | 巡检线程 |
| starting / running | stopping → stopped | 手动停止（manual），或到达 `expires_at`（expired） | 停止接口 / 巡检线程 |
| 非 creating 状态 | deleting → （删除记录） | 删除请求 | 删除接口 |

- 因 STARTUP_TIMEOUT 或 HEALTH_CHECK_FAILED 变为 failed 时，要停止容器但不删除，方便用 `docker logs` 排查，并设置 `stop_reason=failure`。状态保持 failed，不改为 stopped。
- 只处理带本服务 label 的容器。**绝不触碰**已有的 `winarena` 容器或其他容器。

### 6.2 实例锁

在 `manager.py` 中用 `threading.Lock` 保护一个集合 `inflight`，并提供 `try_lock(vm_id)` 和 `unlock(vm_id)`。

- 创建、停止、删除、自动停止，执行前都要先拿到锁，结束后释放。
- 接口在以下情况返回 **VM_BUSY**：状态为 creating，或拿不到锁。
- 状态为 stopping / deleting 但锁空闲，说明上次操作失败或被中断，此时允许重新执行。
- 等待关机期间，只持有这台虚拟机自己的锁。

### 6.3 资源计算

| 项目 | 计算方式 |
| --- | --- |
| CPU 预算 | 逻辑 CPU 数 − `host_reserved_cpu` |
| 内存预算 | 物理内存 − `host_reserved_ram_gb` |
| 单台占用 | CPU = `cpu_cores`；内存 = `ram_gb + vm_overhead_ram_gb` |
| `cpu.total`、`memory_gb.total` | 宿主机物理总量 |
| `cpu.allocated`、`memory_gb.allocated` | 所有 `reserved=1` 记录的占用之和 |
| `cpu.available` | `max(0, CPU 预算 − cpu.allocated)` |
| `memory_gb.available` | `max(0, min(内存预算 − memory_gb.allocated, MemAvailable − host_reserved_ram_gb))`，向下取整。`MemAvailable` 取自 `/proc/meminfo` |
| `vms.running` | `reserved=1` 的记录数 |
| `vms.queued` | queued 的记录数 |

### 6.4 准入检查（只检查队首）

| 情况 | 处理 |
| --- | --- |
| KVM、镜像或基础磁盘已不可用 | 队首直接 failed（HOST_NOT_READY），不阻塞后面的队列 |
| Docker 无法连接 | 继续排队，`queue_reason=docker_unavailable` |
| 满足以下任一条件：`cpu_cores` > `cpu.available`；`ram_gb + overhead` > `memory_gb.available`；Docker 数据目录所在磁盘的剩余空间 < `min_free_disk_gb` | 继续排队，`queue_reason=insufficient_resources` |
| 以上都没有问题 | 领取，开始创建 |

接口返回的 `queue.reason`：队首返回 `queue_reason` 的值，其他排队项一律返回 `waiting_turn`。

Docker 数据目录通过 `client.info()["DockerRootDir"]` 获取，剩余空间用 `shutil.disk_usage` 读取。

## 7. 实现要点

### 7.1 响应与错误处理

- 成功时返回 HTTP 200，响应体直接是数据。
- 失败时抛出 `ApiError(http_status, code, message, details)`，由全局异常处理器转换为 `{"error": {"code", "message", "details"}}`。
- Pydantic 校验错误（包括请求体不是合法 JSON）一律返回 422 VALIDATION_ERROR，`details.fields` 中列出每个出错字段的路径（如 `spec.cpu_cores`）和原因。
- 未捕获的异常返回 500 INTERNAL_ERROR，完整堆栈只写入日志。

### 7.2 创建接口

请求模型各层都设置 `extra="forbid"`。数值字段用 `StrictInt`，且必须大于 0；`software` 必须非空且不重复；`expose` 的两个字段默认都为 `true`；`lifecycle` 的默认值取自 settings。

按以下顺序校验，遇到第一个错误就返回。**校验失败时，不写入任何数据。**

| 顺序 | 检查项 | 错误 |
| --- | --- | --- |
| 1 | 字段、类型、格式 | 422 VALIDATION_ERROR |
| 2 | 「os + 软件集合」能否匹配到某个 profile | 422 UNSUPPORTED_ENVIRONMENT，details 中列出所有支持的组合 |
| 3 | 是否满足该 profile 的最低配置 | 422 BELOW_PROFILE_MINIMUM |
| 4 | 单台占用是否超过预算 | 422 REQUEST_EXCEEDS_CAPACITY |
| 5 | Docker 能否连接（`client.ping()`） | 503 DOCKER_UNAVAILABLE |
| 6 | KVM、镜像、基础磁盘是否可用 | 503 HOST_NOT_READY，`details.missing` 列出缺少的项 |

全部通过后：写入一条 queued 记录，在同一事务内计算 `queue_position`，唤醒巡检线程，然后返回。

### 7.3 停止与删除接口

两个接口都是同步的：在请求线程内先拿锁，等操作完成后再返回。

**停止接口**

| 状态 | 处理 |
| --- | --- |
| 不存在 | 404 VM_NOT_FOUND |
| 正忙（6.2），或为 deleting | 409 VM_BUSY |
| queued | CAS 改为 stopped，`stop_reason=cancelled`，写入 `stopped_at`；不依赖 Docker |
| starting / running / stopping（锁空闲） | Docker 不可用时返回 503。<br>否则先 CAS 改为 stopping，并写入 `stop_reason=manual`，再停止容器。<br>成功：改为 stopped，`reserved=0`，写入 `stopped_at`。<br>失败：返回 500 STOP_FAILED，状态保持 stopping |
| stopped | 原样返回 |
| failed | 若 `reserved=1`，先补做一次停止，再返回当前状态 |

**删除接口**

| 状态 | 处理 |
| --- | --- |
| 不存在 | 200，返回 `deleted` |
| 正忙（6.2） | 409 VM_BUSY |
| queued | 用带 `status='queued'` 条件的 DELETE 删除记录；不依赖 Docker |
| 其他状态（锁空闲） | 有容器且 Docker 不可用时，返回 503。<br>否则先 CAS 改为 deleting，再依次执行：停止容器 → 删除容器 → 删除记录。<br>全部成功：返回 `deleted`。<br>任一步失败：返回 500 DELETE_FAILED，状态保持 deleting |

重试时，如果容器已经不存在，就视为已删除。

### 7.4 创建与删除容器

**创建流程**

1. 调用 `containers.create(...)`，**立即**把返回的 `container_id` 写入数据库；
2. 调用 `start()`；
3. 调用 `reload()`，从 `NetworkSettings.Ports` 读出宿主机端口，转成整数后写入数据库。

**参数对照**：第 1 节命令中的每一项，按下表对应到 SDK 参数。

| 原命令 | SDK 参数 | 说明 |
| --- | --- | --- |
| `-it --rm` | 不使用 | 后台运行，由删除接口负责删除容器 |
| `--name winarena` | `name=f"vm-{vm_id}"` | |
| `-p 8006:8006 -p 3390:3389 -p 5000:5000` | `ports={"5000/tcp": None, "8006/tcp": None, "3389/tcp": None}` | 宿主机端口由 Docker 分配。5000 始终发布；8006、3389 根据 `expose` 决定是否发布 |
| `--platform linux/amd64` | `platform="linux/amd64"` | |
| `--device=/dev/kvm` | `devices=["/dev/kvm:/dev/kvm:rwm"]` | |
| `-e RAM_SIZE=8G -e CPU_CORES=4` | `environment={"RAM_SIZE": f"{ram_gb}G", "CPU_CORES": str(cpu_cores), "BASE_IMG": ..., "BASE_FORMAT": ...}` | 规格取自请求，`BASE_IMG`、`BASE_FORMAT` 取自 profile |
| `-v .../vm/storage/.:/base_storage:ro` | `profile.base_storage → /base_storage`，只读 | |
| `--privileged --cap-add NET_ADMIN` | `privileged=True`，`cap_add=["NET_ADMIN"]` | |
| `--stop-timeout 120` | 停止时调用 `stop(timeout=stop_timeout_seconds)` | |
| `--entrypoint /bin/bash ... -c '...'` | `entrypoint="/bin/bash"`，`command=["/opt/vm-manager/start_windows.sh"]` | 把 `scripts/start_windows.sh` 只读挂载到该路径，脚本内容见 7.5 |
| （新增） | `labels={"vmm.managed": "true", "vmm.vm_id": vm_id}` | 用于识别本服务创建的容器 |
| （新增） | `nano_cpus=cpu_cores * 10**9`；`mem_limit` 与 `memswap_limit` 都设为 `(ram_gb + overhead)` GiB | 限制单台虚拟机的资源 |

- 不设置 auto_remove，也不设置自动重启。
- Docker 客户端：`docker.from_env(timeout=180)`，超时时间要大于停止等待时间。

**停止与删除**

- **停止**：调用 `stop(timeout=stop_timeout_seconds)`，再用 `reload()` 确认容器已退出，才算停止成功。
- **删除**：一律调用 `remove(v=True)`，同时删除容器的匿名卷。镜像可能把 `/storage` 声明为卷，不带 `v=True` 删除会遗留虚拟机磁盘。
- 虚拟机磁盘 `/storage` 不挂载宿主机目录：停止时随容器保留，删除时随容器一起删除。

**接口返回的 access 字段**

- `host` 始终为 `access_host`。
- 三个端口取自数据库。未开放，或容器尚未启动时，为 null。

### 7.5 启动脚本 scripts/start_windows.sh

```bash
echo '1. Construct basic env...'
rm -rf /storage
mkdir -p /storage
cp -n /base_storage/windows* /storage/ || { echo 'ERROR: failed to copy windows* files'; exit 1; }
rm -f /storage/windows*.vars
qemu-img create -f qcow2 -F "$BASE_FORMAT" -b "/base_storage/$BASE_IMG" /storage/data.qcow2 \
  || { echo 'ERROR: failed to create data.qcow2'; exit 1; }
echo '2. Starting VM...'
/usr/bin/tini -s /run/entry.sh &
pid=$!
trap 'kill -TERM "$pid" 2>/dev/null' TERM INT
sleep 1
echo '3. Applying iptables bypass rules...'
iptables -t nat -I PREROUTING 1 -p tcp --dport 8000 -j RETURN
iptables -t nat -I PREROUTING 1 -p tcp --dport 8080 -j RETURN
iptables -t nat -I PREROUTING 1 -p tcp --dport 22 -j RETURN
while kill -0 "$pid" 2>/dev/null; do wait "$pid"; done
```

与第 1 节的原命令相比，这个脚本有以下不同：

- 基础磁盘由环境变量 `BASE_IMG`、`BASE_FORMAT` 指定，不写死为 `data.img`。
- 复制启动文件或创建增量盘失败时，立即退出，不再继续启动虚拟机。`rm -rf /storage` 的退出码不检查，因为 `/storage` 可能是挂载点，目录本身删不掉。
- 输出不再重定向到 `/dev/null`，会进入 `docker logs`。
- 脚本会一直等待 tini 进程，并把停止信号转发给它。虚拟机进程退出后，容器随之退出；停止容器时，Windows 可以正常关机。

如果实测发现容器启动后很快就自行退出，说明 `entry.sh` 不会常驻前台。这时把最后一行改为 `tail -f /dev/null`，并在 TEST_REPORT.md 中记录。

### 7.6 巡检线程

一个后台线程，每隔 `poll_interval_seconds` 执行一轮；收到创建、停止或删除请求时会被唤醒，立即执行。每一轮按以下顺序处理：

1. **列出容器**：按 label 列出本服务的全部容器及运行状态。Docker 无法连接时，跳过所有依赖 Docker 的步骤：既不判为检查失败，也不释放资源。
2. **排队超时**：超过 `queue_deadline` 的 queued 记录，改为 failed（QUEUE_TIMEOUT）。
3. **到期停止**：超过 `expires_at` 的 starting / running 记录，拿到锁后在后台线程中停止，`stop_reason=expired`。
4. **健康检查**：逐台检查 starting / running 的虚拟机。以下各项全部通过才算成功：
   - `GET http://127.0.0.1:<agent 端口><probe_path>` 返回 HTTP 200，且 JSON 中的 `status` 等于 `"Probe successful"`；
   - 开放了 VNC 时，请求 VNC 端口返回 HTTP 2xx 或 3xx；
   - 开放了 RDP 时，RDP 端口能建立 TCP 连接。

   每个请求的超时为 `probe_timeout_seconds`。超时、连接失败、非 200、返回内容不是 JSON、`status` 不符，都算失败。
5. **更新状态**：
   - starting：检查通过 → running；超过启动截止时间 → failed（STARTUP_TIMEOUT）。
   - running：检查失败时 `health_failures + 1`，达到阈值 → failed（HEALTH_CHECK_FAILED）；成功一次就清零。
   - 容器已退出 → failed（CONTAINER_EXITED）。
   - 新变为 failed 且容器仍在运行的，在后台线程中停止容器，但不删除。
6. **补做中断的操作**：锁空闲的 stopping / deleting 记录，以及 `reserved=1` 的 failed 记录，在后台线程中按原来的意图继续执行。同一条记录失败后，至少间隔 60 秒才能重试。
7. **调度队首**：取 seq 最小的 queued 记录做准入检查（6.4），并把结果写入 `queue_reason`。通过后拿锁，CAS 改为 creating，`reserved=1`，写入 `started_at` 和 `expires_at`，然后执行创建流程（7.4）。**每轮最多启动一台。**

后台的停止和删除可以使用 `ThreadPoolExecutor`。巡检中的任何异常都只记录日志，不能让循环退出。

### 7.7 服务重启恢复

在开始接收请求之前执行：

1. **补齐容器信息**：对每条非 queued、非 stopped 的记录，按 label `vmm.vm_id` 找到它的容器。用 inspect 结果补齐 `container_id` 和实际端口，并写入数据库。服务可能在容器已启动、但端口还没写入数据库时退出，这一步就是为了处理这种情况。
2. **按下表处理各状态：**

| 记录状态 | 处理 |
| --- | --- |
| queued | 不变；已超时的由第一轮巡检处理 |
| creating | 容器在运行：改为 starting，`started_at` 不变。<br>否则：改为 failed（PROVISION_FAILED），删除残留的容器，释放资源 |
| starting / running | 容器在运行：保持不变，所有时间字段也不变。<br>否则：改为 failed（CONTAINER_EXITED），`reserved=0` |
| stopping / deleting / `reserved=1` 的 failed | 由第一轮巡检继续处理 |
| stopped | 不处理 |

- 带本服务 label、但找不到对应记录的容器，只记录 warning，不删除。
- 服务正常退出时，只停止巡检线程，不停止任何虚拟机。

## 8. 测试

### 8.1 单元测试（使用假的 Docker 和 HTTP 实现）

只测关键逻辑：

- **校验**：每种错误码各一个用例；软件列表换顺序后仍能匹配；只传 `solidworks-2025` 时返回 UNSUPPORTED_ENVIRONMENT；校验失败后不留下任何记录。
- **排队**：
  - 队首放不下时，后面规格更小的请求也不能启动；
  - 预算只够一台时，两个请求同时提交，最多领取一台；
  - 取消和领取同时发生时，只有一方成功。
- **时间**（使用可控的时钟）：
  - 排队超时变为 QUEUE_TIMEOUT；
  - `expires_at` 从 `started_at` 开始计算；
  - 到期后变为 stopped，且 `stop_reason=expired`。
- **资源**：queued 不占资源；领取后才计入 `allocated`；确认停止后才释放。
- **健康检查**：
  - 非 200、返回内容不是 JSON、`status` 不符、超时，都不能进入 running；
  - 连续失败达到阈值才判为失败，中间成功一次就清零。
- **端口开关**：关闭 VNC 或 RDP 时，不发布对应端口，对应字段为 null，健康检查也跳过这一项。
- **停止与删除**：
  - 各状态下的处理与 7.3 一致；
  - 正忙时返回 VM_BUSY；
  - Docker 不可用时，queued 仍能取消和删除；
  - 删除做到一半失败后，重试能成功；
  - 删除不存在的 ID 返回 200。
- **恢复**：7.7 中的每一条规则，包括「容器已启动、端口未写入」时能补齐端口。

### 8.2 真实环境测试（必须实际执行）

测试前，先记录 CPU、内存、预算、Docker 数据目录所在磁盘的剩余空间、镜像 ID 和 KVM 状况。只操作本次测试创建的容器，不要停止已有的 `winarena` 或其他容器。

分别创建请求 A（`["autocad-2026"]`）和请求 B（三款软件），每个都要验证：

1. 创建接口返回 200 和 `queued`。之后每 5 秒查询一次，直到状态变为 running 或 failed，不能无限等待。
2. 执行 `docker exec <容器> qemu-img info -U /storage/data.qcow2` 检查 backing 文件：A 必须是 `data_v1_autocad_30G.img`，B 必须是 `data.img`。
3. 访问 `http://{host}:{agent_port}/probe`，返回 200 且 `status` 为 `Probe successful`，保存实际返回的 JSON。只在容器内用 curl 访问成功不算通过。
4. `http://{host}:{vnc_port}` 能打开 noVNC 页面，RDP 端口能建立 TCP 连接。如果只检查了 TCP，不能声称已经登录桌面。
5. 核对容器的 CPU 和内存限制。
6. 停止：返回 `stopped`，容器确实已经停止，资源已释放，`docker logs` 中能看到虚拟机的输出；重复停止仍返回 200。记录停止的实际耗时，用于确认停止信号已经转发给虚拟机。
7. 删除：返回 `deleted`，容器及其匿名卷都已删除（`docker volume ls` 中不再出现），再次查询返回 404；重复删除仍返回 200。

资源允许时，让 A、B 同时运行，验证两者的端口不同、探活地址都可用。资源不够时，如实记录缺少多少资源，并把这一项标为未完成；不能降低规格，也不能关闭别人的虚拟机。

以下场景也必须实测：

- 预算只够一台时，第二台进入排队；第一台停止后，第二台自动开始创建。
- 关闭 VNC 或 RDP 时，对应端口确实没有发布。
- `timeout_minutes=1`：到期后自动变为 stopped，`stop_reason=expired`。
- `queue_timeout_minutes=1` 且资源一直被占满：变为 failed（QUEUE_TIMEOUT）。
- 使用一个测试 profile，把 `probe_path` 设为不存在的路径，并缩短启动超时：结果为 failed（STARTUP_TIMEOUT），容器已停止但仍保留。
- 存在 running 和 queued 的虚拟机时重启服务：虚拟机 ID、容器、端口、到期时间都不变，排队顺序也不变，且不会出现重复创建的虚拟机。

### 8.3 TEST_REPORT.md

如实记录以下内容：

- A、B 的请求、HTTP 状态码、vm_id、端口、probe 返回内容、backing 检查结果；
- 停止、删除、排队、超时、恢复的结果，以及停止耗时；
- 资源接口的返回值；
- 关键错误码的测试结果。

真实测试和 mock 测试分开记录。**不能用 mock 测试代替真实虚拟机的验收。** 缺少权限或资源时，只完成能做的部分，并列出阻塞项。测试结束后，删除本次创建的全部虚拟机。

**完成标准**：两个基础磁盘都能成功启动，probe 通过；停止、删除、排队、超时、资源查询、服务重启恢复，以及关键错误码，全部经过实测并通过。 