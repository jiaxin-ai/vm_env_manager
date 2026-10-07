# VM Manager API v1

本文档定义 VM Manager V1 的接口规范。

服务地址：`<ip>:<port>`。请求和响应均为 JSON。

成功时 HTTP 状态码为 200，响应体为数据；

失败时 HTTP 状态码为 4xx / 5xx，响应体为 `{"error": {"code": ..., "message": ..., "details": ...}}`。

## 接口总览

V1 共6个基础接口：


| 序号  | 接口名称    | 请求方法   | 请求路径                   |
| --- | ------- | ------ | ---------------------- |
| 1   | 查询宿主机资源 | GET    | `/v1/host/resources`   |
| 2   | 创建虚拟机   | POST   | `/v1/vms`              |
| 3   | 查询虚拟机列表 | GET    | `/v1/vms`              |
| 4   | 查询虚拟机详情 | GET    | `/v1/vms/{vm_id}`      |
| 5   | 停止虚拟机   | POST   | `/v1/vms/{vm_id}/stop` |
| 6   | 删除虚拟机   | DELETE | `/v1/vms/{vm_id}`      |


---

## 1. 查询宿主机资源

**接口描述**

查询总共、已占用和当前可分配给新虚拟机的 CPU 和内存资源，以及当前虚拟机数量。

- 每台虚拟机实际占用的内存为 `ram_gb + 1` GB，其中 1 GB 是虚拟化开销。
- 队首请求所需的资源不超过可用量时，会自动开始创建；否则继续排队。
- 返回的是查询时刻的数据，不会预留资源。

**请求地址**

`GET http://<ip>:<port>/v1/host/resources`

**请求参数**

无

**响应参数**


| 参数名称                | 类型      | 描述                                                                           |
| ------------------- | ------- | ---------------------------------------------------------------------------- |
| cpu.total           | Integer | 宿主机 CPU 总核数                                                                  |
| cpu.allocated       | Integer | 已被占用的核数                                                                      |
| cpu.available       | Integer | 可分配给新虚拟机的核数。宿主机自身需要预留一部分，因此小于 `total − allocated`                            |
| memory_gb.total     | Integer | 宿主机内存总量（GB）                                                                  |
| memory_gb.allocated | Integer | 已被占用的内存（GB），每台按 `ram_gb + 1` 计算                                              |
| memory_gb.available | Integer | 可分配给新虚拟机的内存（GB）。宿主机自身需要预留一部分，因此小于 `total − allocated`；宿主机实际空闲内存更少时，以实际空闲内存为准 |
| vms.running         | Integer | 正在占有资源的虚拟机数                                                                  |
| vms.queued          | Integer | 排队中的虚拟机数                                                                     |


**调用示例**

请求示例：

```bash
curl http://<ip>:<port>/v1/host/resources
```

成功响应示例：

```json
{
  "cpu": {"total": 32, "allocated": 8, "available": 22},
  "memory_gb": {"total": 128, "allocated": 18, "available": 106},
  "vms": {"running": 2, "queued": 0}
}
```

失败响应示例：

```json
{"error": {"code": "INTERNAL_ERROR", "message": "服务内部错误", "details": null}}
```

**返回码**


| HTTP 状态码 | code           | 说明     | details |
| -------- | -------------- | ------ | ------- |
| 200      | —              | 成功     | —       |
| 500      | INTERNAL_ERROR | 服务内部错误 | null    |


---



## 2. 创建虚拟机

**接口描述**

创建一台虚拟机。本接口为异步接口：

- 请求通过校验后进入排队并立即返回，此时 `status` 固定为 `queued`。
- 资源足够时，几秒内会自动开始创建；资源不足时继续排队，按提交的先后顺序依次创建。

提交后，轮询「查询虚拟机详情」，`status` 变为 `running` 表示可用。以下三种情况会自动结束：

- 排队超过 `queue_timeout_minutes` 仍未开始创建：变为 `failed`（QUEUE_TIMEOUT）。
- 开始创建后 15 分钟仍未就绪：变为 `failed`（STARTUP_TIMEOUT）。
- 开始创建后超过 `timeout_minutes`：自动停止，变为 `stopped`，数据保留。

每次调用都会新建一台虚拟机。如果请求网络超时，请先查询列表确认是否已创建，不要直接重试。

**请求地址**

`POST http://<ip>:<port>/v1/vms`

**请求参数**


| 参数名称                            | 位置   | 类型              | 必填  | 描述                                     | 示例值              |
| ------------------------------- | ---- | --------------- | --- | -------------------------------------- | ---------------- |
| os                              | Body | String          | 是   | 操作系统，目前仅支持 `windows11`                 | windows11        |
| software                        | Body | Array of String | 是   | 预装软件列表，必须与下方「可选环境」表中某一行的软件组合完全一致（顺序不限） | ["autocad-2026"] |
| spec.cpu_cores                  | Body | Integer         | 是   | CPU 核数，不能低于所选环境的最低要求                   | 4                |
| spec.ram_gb                     | Body | Integer         | 是   | 内存（GB），不能低于所选环境的最低要求                   | 8                |
| expose.vnc                      | Body | Boolean         | 否   | 是否开放 VNC 网页，默认 `true`                  | true             |
| expose.rdp                      | Body | Boolean         | 否   | 是否开放 RDP，默认 `true`                     | true             |
| lifecycle.timeout_minutes       | Body | Integer         | 否   | 最长运行时长（分钟），从开始创建时算起，到期自动停止，默认 120      | 120              |
| lifecycle.queue_timeout_minutes | Body | Integer         | 否   | 最长排队时长（分钟），从提交时算起，到期仍未开始创建则失败，默认 30    | 30               |
| name                            | Body | String          | 否   | 自定义名称，1–64 个字符                         | test-01          |


不接受表中以外的字段。Agent 接口端口始终开放，不受 `expose` 控制。

可选环境：


| os        | software                                             | 包含的软件                                      | 最低 CPU 核数 | 最低内存（GB） |
| --------- | ---------------------------------------------------- | ------------------------------------------ | --------- | -------- |
| windows11 | ["autocad-2026"]                                     | AutoCAD 2026                               | 4         | 8        |
| windows11 | ["autocad-2026", "solidworks-2025", "ansys-2024-r1"] | AutoCAD 2026、SolidWorks 2025、ANSYS 2024 R1 | 4         | 8        |


**响应参数**


| 参数名称           | 类型      | 描述              |
| -------------- | ------- | --------------- |
| vm_id          | String  | 虚拟机 ID          |
| status         | String  | 固定为 `queued`    |
| queue_position | Integer | 提交时的排队序号，从 1 开始 |


**调用示例**

请求示例：

```bash
curl -X POST http://<ip>:<port>/v1/vms \
  -H "Content-Type: application/json" \
  -d '{"os": "windows11", "software": ["autocad-2026"], "spec": {"cpu_cores": 4, "ram_gb": 8}, "expose": {"vnc": true, "rdp": true}, "lifecycle": {"timeout_minutes": 120, "queue_timeout_minutes": 30}, "name": "test-01"}'
```

成功响应示例：

```json
{"vm_id": "a1b2c3d4", "status": "queued", "queue_position": 1}
```

失败响应示例：

```json
{
  "error": {
    "code": "BELOW_PROFILE_MINIMUM",
    "message": "资源规格低于环境最低配置",
    "details": {
      "requested": {"cpu_cores": 2, "ram_gb": 4},
      "minimum": {"cpu_cores": 4, "ram_gb": 8}
    }
  }
}
```

**返回码**


| HTTP 状态码 | code                     | 说明                            | details                                                               |
| -------- | ------------------------ | ----------------------------- | --------------------------------------------------------------------- |
| 200      | —                        | 成功，已进入排队                      | —                                                                     |
| 422      | VALIDATION_ERROR         | 缺少必填字段、类型错误、取值非法、软件重复，或出现未知字段 | `fields`：出错字段列表，如 `[{"field": "spec.cpu_cores", "reason": "必须为正整数"}]` |
| 422      | UNSUPPORTED_ENVIRONMENT  | 不支持的操作系统或软件组合                 | `requested`：请求的 os 和 software；`supported`：支持的全部组合                     |
| 422      | BELOW_PROFILE_MINIMUM    | 资源规格低于环境最低配置                  | `requested`、`minimum`                                                 |
| 422      | REQUEST_EXCEEDS_CAPACITY | 资源规格超出宿主机总容量，即使宿主机空闲也无法创建     | `requested`；`required`：含开销的实际占用；`capacity`：宿主机可分配总量                   |
| 503      | DOCKER_UNAVAILABLE       | Docker 服务不可用                  | null                                                                  |
| 503      | HOST_NOT_READY           | 宿主机环境未就绪                      | `missing`：缺少的项，可能为 `kvm`、`image`、`base_disk`                          |
| 500      | INTERNAL_ERROR           | 服务内部错误                        | null                                                                  |


---



## 3. 查询虚拟机列表

**接口描述**

按提交时间倒序，返回全部未删除的虚拟机。

**请求地址**

`GET http://<ip>:<port>/v1/vms`

**请求参数**


| 参数名称   | 位置    | 类型     | 必填  | 描述                      | 示例值              |
| ------ | ----- | ------ | --- | ----------------------- | ---------------- |
| status | Query | String | 否   | 按状态过滤，多个状态用逗号分隔；不传时返回全部 | running,starting |


**响应参数**


| 参数名称                                    | 类型              | 描述                                                                                                                                                          |
| --------------------------------------- | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| total                                   | Integer         | 虚拟机数量                                                                                                                                                       |
| items                                   | Array           | 虚拟机列表                                                                                                                                                       |
| items[].vm_id                           | String          | 虚拟机 ID                                                                                                                                                      |
| items[].name                            | String          | 名称；未指定时为 null                                                                                                                                               |
| items[].status                          | String          | 状态：`queued` 排队中、`creating` 创建中、`starting` 启动中、`running` 运行中、`stopping` 停止中、`stopped` 已停止、`deleting` 删除中、`failed` 失败                                         |
| items[].os                              | String          | 操作系统                                                                                                                                                        |
| items[].software                        | Array of String | 预装软件列表                                                                                                                                                      |
| items[].spec.cpu_cores                  | Integer         | CPU 核数                                                                                                                                                      |
| items[].spec.ram_gb                     | Integer         | 内存（GB）                                                                                                                                                      |
| items[].expose.vnc                      | Boolean         | 是否开放 VNC 网页                                                                                                                                                 |
| items[].expose.rdp                      | Boolean         | 是否开放 RDP                                                                                                                                                    |
| items[].lifecycle.timeout_minutes       | Integer         | 最长运行时长（分钟）                                                                                                                                                  |
| items[].lifecycle.queue_timeout_minutes | Integer         | 最长排队时长（分钟）                                                                                                                                                  |
| items[].container_id                    | String          | Docker 容器 ID；容器创建前为 null                                                                                                                                    |
| items[].access.host                     | String          | 宿主机地址，以下端口都在这个地址上                                                                                                                                           |
| items[].access.vnc_port                 | Integer         | VNC 网页端口，访问 `http://{host}:{vnc_port}`；未开放或容器启动前为 null                                                                                                      |
| items[].access.rdp_port                 | Integer         | RDP 端口，连接 `{host}:{rdp_port}`；未开放或容器启动前为 null                                                                                                               |
| items[].access.agent_port               | Integer         | Agent 接口端口，访问 `http://{host}:{agent_port}`，探活地址为 `http://{host}:{agent_port}/probe`；容器启动前为 null                                                             |
| items[].queue                           | Object          | 排队信息；仅 `queued` 时有值，其他状态为 null                                                                                                                              |
| items[].queue.position                  | Integer         | 排队序号，从 1 开始                                                                                                                                                 |
| items[].queue.reason                    | String          | 等待原因：`waiting_turn` 前面还有请求、`insufficient_resources` 宿主机资源不足、`docker_unavailable` Docker 暂时不可用                                                               |
| items[].queue.expires_at                | String          | 排队截止时间                                                                                                                                                      |
| items[].created_at                      | String          | 提交时间（ISO 8601，UTC）                                                                                                                                          |
| items[].started_at                      | String          | 开始创建的时间；排队中为 null                                                                                                                                           |
| items[].expires_at                      | String          | 到期自动停止的时间；排队中为 null                                                                                                                                         |
| items[].stopped_at                      | String          | 停止时间；未停止时为 null                                                                                                                                             |
| items[].stop_reason                     | String          | 停止原因：`manual` 手动停止、`cancelled` 排队中被取消、`expired` 到期自动停止、`failure` 失败后自动停止；未停止时为 null                                                                         |
| items[].error                           | Object          | 失败信息；没有失败时为 null                                                                                                                                            |
| items[].error.code                      | String          | `QUEUE_TIMEOUT` 排队超时、`HOST_NOT_READY` 宿主机环境未就绪、`PROVISION_FAILED` 创建失败、`STARTUP_TIMEOUT` 启动超时、`CONTAINER_EXITED` 容器异常退出、`HEALTH_CHECK_FAILED` 运行中连续 3 次检查失败 |
| items[].error.message                   | String          | 失败原因说明                                                                                                                                                      |


**调用示例**

请求示例：

```bash
curl "http://<ip>:<port>/v1/vms?status=running,starting"
```

成功响应示例：

```json
{
  "total": 1,
  "items": [
    {
      "vm_id": "a1b2c3d4",
      "name": "test-01",
      "status": "running",
      "os": "windows11",
      "software": ["autocad-2026"],
      "spec": {"cpu_cores": 4, "ram_gb": 8},
      "expose": {"vnc": true, "rdp": true},
      "lifecycle": {"timeout_minutes": 120, "queue_timeout_minutes": 30},
      "container_id": "3f9c2e7a1b5d...",
      "access": {
        "host": "10.0.0.10",
        "vnc_port": 32781,
        "rdp_port": 32782,
        "agent_port": 32783
      },
      "queue": null,
      "created_at": "2026-10-07T10:00:00Z",
      "started_at": "2026-10-07T10:00:05Z",
      "expires_at": "2026-10-07T12:00:05Z",
      "stopped_at": null,
      "stop_reason": null,
      "error": null
    }
  ]
}
```

失败响应示例：

```json
{"error": {"code": "VALIDATION_ERROR", "message": "参数错误", "details": {"fields": [{"field": "status", "reason": "取值错误：runing"}]}}}
```

**返回码**


| HTTP 状态码 | code             | 说明          | details         |
| -------- | ---------------- | ----------- | --------------- |
| 200      | —                | 成功          | —               |
| 422      | VALIDATION_ERROR | status 取值错误 | `fields`：出错字段列表 |
| 500      | INTERNAL_ERROR   | 服务内部错误      | null            |


---



## 4. 查询虚拟机详情

**接口描述**

查询单台虚拟机的状态、访问地址、时间信息和失败原因。

- `status` 为 `running` 表示虚拟机可用：容器正在运行，探活地址 `http://{host}:{agent_port}/probe` 返回成功，已开放的 VNC 和 RDP 可以连接。
- 运行期间系统会持续检查，连续 3 次检查失败则变为 `failed`（HEALTH_CHECK_FAILED），并自动停止。
- 停止或失败后，仍会返回原来分配的地址供排查使用，但不保证这些地址仍可访问。

**请求地址**

`GET http://<ip>:<port>/v1/vms/{vm_id}`

**请求参数**


| 参数名称  | 位置   | 类型     | 必填  | 描述     | 示例值      |
| ----- | ---- | ------ | --- | ------ | -------- |
| vm_id | Path | String | 是   | 虚拟机 ID | a1b2c3d4 |


**响应参数**


| 参数名称                            | 类型              | 描述                                                                                                                                                          |
| ------------------------------- | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| vm_id                           | String          | 虚拟机 ID                                                                                                                                                      |
| name                            | String          | 名称；未指定时为 null                                                                                                                                               |
| status                          | String          | 状态：`queued` 排队中、`creating` 创建中、`starting` 启动中、`running` 运行中、`stopping` 停止中、`stopped` 已停止、`deleting` 删除中、`failed` 失败                                         |
| os                              | String          | 操作系统                                                                                                                                                        |
| software                        | Array of String | 预装软件列表                                                                                                                                                      |
| spec.cpu_cores                  | Integer         | CPU 核数                                                                                                                                                      |
| spec.ram_gb                     | Integer         | 内存（GB）                                                                                                                                                      |
| expose.vnc                      | Boolean         | 是否开放 VNC 网页                                                                                                                                                 |
| expose.rdp                      | Boolean         | 是否开放 RDP                                                                                                                                                    |
| lifecycle.timeout_minutes       | Integer         | 最长运行时长（分钟）                                                                                                                                                  |
| lifecycle.queue_timeout_minutes | Integer         | 最长排队时长（分钟）                                                                                                                                                  |
| container_id                    | String          | Docker 容器 ID；容器创建前为 null                                                                                                                                    |
| access.host                     | String          | 宿主机地址，以下端口都在这个地址上                                                                                                                                           |
| access.vnc_port                 | Integer         | VNC 网页端口，访问 `http://{host}:{vnc_port}`；未开放或容器启动前为 null                                                                                                      |
| access.rdp_port                 | Integer         | RDP 端口，连接 `{host}:{rdp_port}`；未开放或容器启动前为 null                                                                                                               |
| access.agent_port               | Integer         | Agent 接口端口，访问 `http://{host}:{agent_port}`，探活地址为 `http://{host}:{agent_port}/probe`；容器启动前为 null                                                             |
| queue                           | Object          | 排队信息；仅 `queued` 时有值，其他状态为 null                                                                                                                              |
| queue.position                  | Integer         | 排队序号，从 1 开始                                                                                                                                                 |
| queue.reason                    | String          | 等待原因：`waiting_turn` 前面还有请求、`insufficient_resources` 宿主机资源不足、`docker_unavailable` Docker 暂时不可用                                                               |
| queue.expires_at                | String          | 排队截止时间，到期仍未开始创建则变为 `failed`                                                                                                                                 |
| created_at                      | String          | 提交时间（ISO 8601，UTC）                                                                                                                                          |
| started_at                      | String          | 开始创建的时间；排队中为 null                                                                                                                                           |
| expires_at                      | String          | 到期自动停止的时间，等于 `started_at` 加上 `timeout_minutes`；排队中为 null                                                                                                    |
| stopped_at                      | String          | 停止时间；未停止时为 null                                                                                                                                             |
| stop_reason                     | String          | 停止原因：`manual` 手动停止、`cancelled` 排队中被取消、`expired` 到期自动停止、`failure` 失败后自动停止；未停止时为 null                                                                         |
| error                           | Object          | 失败信息；没有失败时为 null                                                                                                                                            |
| error.code                      | String          | `QUEUE_TIMEOUT` 排队超时、`HOST_NOT_READY` 宿主机环境未就绪、`PROVISION_FAILED` 创建失败、`STARTUP_TIMEOUT` 启动超时、`CONTAINER_EXITED` 容器异常退出、`HEALTH_CHECK_FAILED` 运行中连续 3 次检查失败 |
| error.message                   | String          | 失败原因说明                                                                                                                                                      |


**调用示例**

请求示例：

```bash
curl http://<ip>:<port>/v1/vms/a1b2c3d4
```

成功响应示例：

```json
{
  "vm_id": "a1b2c3d4",
  "name": "test-01",
  "status": "running",
  "os": "windows11",
  "software": ["autocad-2026"],
  "spec": {"cpu_cores": 4, "ram_gb": 8},
  "expose": {"vnc": true, "rdp": true},
  "lifecycle": {"timeout_minutes": 120, "queue_timeout_minutes": 30},
  "container_id": "3f9c2e7a1b5d...",
  "access": {
    "host": "10.0.0.10",
    "vnc_port": 32781,
    "rdp_port": 32782,
    "agent_port": 32783
  },
  "queue": null,
  "created_at": "2026-10-07T10:00:00Z",
  "started_at": "2026-10-07T10:00:05Z",
  "expires_at": "2026-10-07T12:00:05Z",
  "stopped_at": null,
  "stop_reason": null,
  "error": null
}
```

排队中的响应示例（只列出与上面不同的字段）：

```json
{
  "status": "queued",
  "container_id": null,
  "access": {"host": "10.0.0.10", "vnc_port": null, "rdp_port": null, "agent_port": null},
  "queue": {"position": 2, "reason": "waiting_turn", "expires_at": "2026-10-07T10:30:00Z"},
  "started_at": null,
  "expires_at": null
}
```

失败响应示例：

```json
{"error": {"code": "VM_NOT_FOUND", "message": "虚拟机不存在", "details": {"vm_id": "a1b2c3d4"}}}
```

**返回码**


| HTTP 状态码 | code           | 说明     | details |
| -------- | -------------- | ------ | ------- |
| 200      | —              | 成功     | —       |
| 404      | VM_NOT_FOUND   | 虚拟机不存在 | `vm_id` |
| 500      | INTERNAL_ERROR | 服务内部错误 | null    |


---



## 5. 停止虚拟机

**接口描述**

停止虚拟机并释放 CPU 和内存，磁盘数据保留。

v1 不支持重新启动已停止的虚拟机，保留的数据仅用于排查问题；不再需要时，请调用「删除虚拟机」释放磁盘。

本接口为同步接口，需要等待 Windows 关机，最长约 2 分钟，客户端超时请设为 180 秒以上。


| 当前状态                       | 结果                                                     |
| -------------------------- | ------------------------------------------------------ |
| queued                     | 取消排队；`status` 变为 `stopped`，`stop_reason` 为 `cancelled` |
| starting / running         | 关机后停止；`status` 变为 `stopped`，`stop_reason` 为 `manual`   |
| stopped / failed           | 已经处于停止状态，直接返回当前状态                                      |
| stopping，且上次停止失败           | 重新执行停止                                                 |
| creating / deleting，或正在停止中 | 返回 409 VM_BUSY，请稍后重试                                   |


**请求地址**

`POST http://<ip>:<port>/v1/vms/{vm_id}/stop`

**请求参数**


| 参数名称  | 位置   | 类型     | 必填  | 描述     | 示例值      |
| ----- | ---- | ------ | --- | ------ | -------- |
| vm_id | Path | String | 是   | 虚拟机 ID | a1b2c3d4 |


**响应参数**


| 参数名称        | 类型     | 描述                                                                                         |
| ----------- | ------ | ------------------------------------------------------------------------------------------ |
| vm_id       | String | 虚拟机 ID                                                                                     |
| status      | String | 当前状态：`stopped` 已停止，或 `failed` 失败（失败的虚拟机已被自动停止）                                             |
| stop_reason | String | 停止原因：`manual` 手动停止、`cancelled` 排队中被取消、`expired` 到期自动停止、`failure` 失败后自动停止；从未运行过的失败虚拟机为 null |
| stopped_at  | String | 停止时间（ISO 8601，UTC）；从未运行过的失败虚拟机为 null                                                       |


**调用示例**

请求示例：

```bash
curl -X POST http://<ip>:<port>/v1/vms/a1b2c3d4/stop
```

成功响应示例：

```json
{"vm_id": "a1b2c3d4", "status": "stopped", "stop_reason": "manual", "stopped_at": "2026-10-07T11:00:30Z"}
```

失败响应示例：

```json
{"error": {"code": "VM_BUSY", "message": "虚拟机正在创建中，请稍后重试", "details": {"status": "creating"}}}
```

**返回码**


| HTTP 状态码 | code               | 说明                                      | details       |
| -------- | ------------------ | --------------------------------------- | ------------- |
| 200      | —                  | 成功                                      | —             |
| 404      | VM_NOT_FOUND       | 虚拟机不存在                                  | `vm_id`       |
| 409      | VM_BUSY            | 虚拟机正在创建、停止或删除中                          | `status`：当前状态 |
| 503      | DOCKER_UNAVAILABLE | Docker 服务不可用，无法停止容器（排队中的虚拟机不受影响，可以正常取消） | null          |
| 500      | STOP_FAILED        | 停止未完成，可再次调用重试                           | `vm_id`       |
| 500      | INTERNAL_ERROR     | 服务内部错误                                  | null          |


---



## 6. 删除虚拟机

**接口描述**

删除虚拟机的容器、磁盘数据和记录，并释放全部资源。删除后，再查询该虚拟机会返回 404。

本接口为同步接口：运行中的虚拟机需要先关机，最长约 2 分钟，客户端超时请设为 180 秒以上。


| 当前状态                        | 结果                   |
| --------------------------- | -------------------- |
| queued                      | 取消排队并删除记录            |
| starting / running          | 关机后删除                |
| stopped / failed            | 直接删除                 |
| stopping / deleting，且上次操作失败 | 重新执行删除               |
| creating，或正在停止、删除中          | 返回 409 VM_BUSY，请稍后重试 |
| 不存在（包括已删除）                  | 同样返回成功               |


**请求地址**

`DELETE http://<ip>:<port>/v1/vms/{vm_id}`

**请求参数**


| 参数名称  | 位置   | 类型     | 必填  | 描述     | 示例值      |
| ----- | ---- | ------ | --- | ------ | -------- |
| vm_id | Path | String | 是   | 虚拟机 ID | a1b2c3d4 |


**响应参数**


| 参数名称   | 类型     | 描述            |
| ------ | ------ | ------------- |
| vm_id  | String | 虚拟机 ID        |
| status | String | 固定为 `deleted` |


**调用示例**

请求示例：

```bash
curl -X DELETE http://<ip>:<port>/v1/vms/a1b2c3d4
```

成功响应示例：

```json
{"vm_id": "a1b2c3d4", "status": "deleted"}
```

失败响应示例：

```json
{"error": {"code": "DELETE_FAILED", "message": "删除未完成，请重试", "details": {"vm_id": "a1b2c3d4"}}}
```

**返回码**


| HTTP 状态码 | code               | 说明                                      | details       |
| -------- | ------------------ | --------------------------------------- | ------------- |
| 200      | —                  | 成功                                      | —             |
| 409      | VM_BUSY            | 虚拟机正在创建、停止或删除中                          | `status`：当前状态 |
| 503      | DOCKER_UNAVAILABLE | Docker 服务不可用，无法删除容器（排队中的虚拟机不受影响，可以正常删除） | null          |
| 500      | DELETE_FAILED      | 删除未完成，记录仍保留，可再次调用重试                     | `vm_id`       |
| 500      | INTERNAL_ERROR     | 服务内部错误                                  | null          |

