# 策略服务 HTTP 接口（Model API）

Oct 22, 2026（本次更新：图像传输新增 `extra_info.image_format`，可发原始像素；在线模型都改为服务端 RTC。Oct 22 下午：服务端代码新增 UBT Agent RTC API 的两种原始图像格式——`raw:uint8:` 前缀 JSON 与 `MIMG` 二进制请求体，Oct 22 下午已部署到两个在线端点，见 §3.2。Oct 22 晚：RTC 前缀改由客户端给出——客户端在 `extra_info.rtc` 里发它已承诺执行的 N 行，服务端原样作为条件，不再估计延迟，已上线，见 §6.1。Oct 23 晚：回复新增 `extra_info.yolo`——每条手臂的“是否夹着箱子 / 当前技能 / 技能内进度”，来自腕部 YOLO 而不是策略模型，见 §5.5。Oct 24：腕部 YOLO 换成统一模型，`yolo` 每条手臂新增夹爪与箱子的关系 `contact`（hold / hit / other）及三类概率 `contact_probs`，见 §5.5）

一个网关按 URL 前缀转发到各个模型服务：`http://<gateway>/<端点>/...`。所有端点的请求/回复格式**相同**，客户端换模型只改 URL 前缀。

- 协议：`manifold_msg` 1.4.1（相对 1.4.0 只多了回复里的 `extra_info`）。图像格式由请求的 `extra_info.image_format` 选择，协议库本身不变（§3.1）。
- 健康检查：`GET /<端点>/` → 200；后端停止时返回 502。
- 图像传输的设计、测速和取舍见 [robot-image-transport](../robot-image-transport/README.md)。

## 1. 当前在线的模型

GPU 为 24 GB 4090，同时最多 2 个服务在线。端点名另行告知。

| 模型 | 手臂 / 状态维 | 动作 | 目标框叠加 | 回复 `extra_info` | 图像格式 |
|---|---|---|---|---|---|
| 双臂 pi05 + 训练式 RTC | 双臂，16D（每臂 7 关节 + 夹爪） | 左右各 50×8 | 两个腕部各画面积最大的框（YOLO，不跟踪） | `target_bbox` = `[左腕框, 右腕框]`；`hold_prob` / `holding`（§5.1）；`yolo` 含 `left`、`right`（§5.5） | 见 §3 一览（四种都支持） |
| 左臂 pi05 + 训练式 RTC | 左臂，8D | 左臂 50×8；右臂保持观测值 | 左腕目标框（YOLO + 跟踪） | `target_bbox` = 左腕框；`yolo` 含 `left`（§5.5） | 见 §3 一览（四种都支持） |

- 两者都**没有**空闲超时重置（§6）。回复里的技能、进度、是否夹持一律看 `yolo`（§5.5）。
- prompt 由服务端固定为训练时的值（形如 `pick the red and white box, place it on the tray operator=<name>`），请求里的 `prompt` 被忽略。
- 原始像素 base64（`extra_info.image_format = "pixel/binary"`）只有 Oct 22 及之后部署的服务支持；更早的服务收到它返回 HTTP 500（§3.1）。

## 2. 调用流程

```
POST /<端点>/submit          # 提交观测，立即返回 {"status": "success", ...}
GET  /<端点>/result/<ts>     # 轮询：202 = 处理中，200 = 完成（含动作）
```

- `<ts>` = 提交时 `timestamp` 的原值。
- 延迟（4090，单请求，经网关）：JPEG 约 0.10 s；`pixel/binary` 约 0.13 s（多出的是 3.8 MB 请求体的传输与解析）。
- 服务（重）启后会自动预热；没有预热时第一次请求要编译模型，需 15~50 s。

## 3. 请求体（`POST /submit`，JSON）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `timestamp` | float | ✔ | 请求 ID，**同时是这帧观测的采集时间（秒）**。正常单调递增；**比上一次小 = 新回合**（§6）。不带 `extra_info.rtc` 的旧客户端，服务端用相邻两次的差值估计 RTC 前缀（§6.2），所以必须是真实时间、单位秒 |
| `prompt` | string | ✖ | 服务端忽略 |
| `img_front` / `img_left` / `img_right` | list[frame] | ✔ | 头部 / 左腕 / 右腕相机，服务端只取最后一帧。帧格式见 §3.1 |
| `left_arm_joint_state` / `right_arm_joint_state` | list[list[8]] | ✔ | 7 关节（弧度，绝对值）+ 1 夹爪宽度；只取最后一帧。单臂模型用右臂当前值保持右臂 |
| `img_*_depth` / `*_joint_velocity` / `*_joint_effort` / `*_end_pose` | list | ✖ | 空表即可 |
| `is_euler` | bool | ✖ | 默认 false |
| `extra_info` | object | ✖ | 上行自由 JSON（对象或其 JSON 字符串）。服务端读其中的 `image_format`（§3.1）、`target_bbox`（§5.2）和 `rtc`（§6.1），其余键接收但不使用 |

### 图像传输方式一览

**base64 JSON 和二进制都支持，并存，客户端任选。** 前三种都是普通的 `application/json` 请求、图像为 base64 字符串；只有最后一种是二进制请求体。

| 方式 | 请求体 | 帧的 `data` | 怎么声明 | 状态 |
|---|---|---|---|---|
| JPEG（默认） | JSON | base64(JPEG) | 什么都不用加 | ✅ 已上线 |
| 原始像素 base64 | JSON | base64(原始像素) | `extra_info.image_format = "pixel/binary"`（名字里的 binary 指像素未压缩，传输仍是 base64 JSON） | ✅ 已上线 |
| 原始像素 base64（UBT Agent RTC API 写法） | JSON | `raw:uint8:<base64>` / `raw:uint16:<base64>` | 前缀本身，逐帧 | 已上线（§3.2a） |
| 原始像素二进制 | `MIMG` 二进制 | 无（像素字节接在 header 后） | `Content-Type: application/vnd.manifold.observation.v1` | 已上线（§3.2b） |

- 两种原始像素 base64 写法解出的图像完全相同；新客户端推荐用 `raw:uint8:` 前缀（自描述、与 UBT Agent RTC API 一致）。
- 二进制比 base64 JSON 请求体小约 25%，省掉 base64 编解码；不在乎这点开销就用 base64。

### 3.1 图像帧格式（`extra_info.image_format`）

帧结构都是 `{"data": ..., "shape": [...]}`；`data` 的含义由 `extra_info.image_format` 决定，**对本次请求的所有相机生效**。

| `extra_info.image_format` | 帧的 `data` | `shape` |
|---|---|---|
| 不带 / `""` / `"jpeg/base64"`（默认） | base64(JPEG 字节)，**BGR**，原始分辨率（当前 640×480） | 原样传，服务端不依赖 |
| `"pixel/binary"` | base64(原始像素字节)：彩色为 **BGR** uint8 `[H, W, 3]`；深度为小端 uint16 `[H, W]`；行优先、连续、无行尾填充（`np.ascontiguousarray(img).tobytes()`） | **必须**是图像本身的尺寸；字节数必须正好等于 H×W×3（深度 H×W×2） |

- 不带 `image_format` 的旧客户端行为完全不变。
- `pixel/binary`：**客户端不缩放、不补边、不转颜色**，直接发相机原图。缩放、补边、裁剪和 BGR→RGB 全部由服务端按各模型的训练方式完成。
- 现有客户端库会先把图像缩放到 `/info` 给的尺寸再编码；发原始像素时要绕开这一步。
- 出错即拒绝，服务端不猜格式：`image_format` 不认识（如 `"png"`）、`shape` 不对、字节数与 `shape` 不符、base64 非法，`/submit` 都返回 **HTTP 500** 和 `{"status": "error", "message": "<原因>"}`，该请求不进入推理。
- 旧服务端收到 `pixel/binary` 帧也返回 500。客户端可以据此回退：收到 500 就去掉 `image_format`、改回 JPEG 重发。
- 取舍：原始像素在机器人端几乎不花时间（JPEG q100 编码实测约 160 ms），但每次请求约 2.8 MB（JSON 体 3.8 MB），**要求机器人到服务器全程千兆有线**；百兆或 Wi‑Fi 下会比 JPEG 更慢。

### 3.2 UBT Agent RTC API 的图像格式（已上线）

服务端代码（manifold_model `a7c8c96`）另外接受 UBT Agent RTC API 定义的两种原始图像格式。Oct 22 下午已部署到两个在线端点（代码 `5f56636`）；`/info` 返回 `raw_image_protocol: 1` 与 `binary_image_protocol: 1` 即表示支持。更早部署的服务没有这两个字段。

像素约定与 §3.1 的 `pixel/binary` 相同：彩色 BGR uint8 `[H, W, 3]`，深度小端 uint16 `[H, W]`，行优先、原始尺寸，客户端不缩放、不补边。

**(a) raw Base64 JSON**：请求仍是 `application/json`，每帧 `data` 带前缀，**不需要** `extra_info.image_format`，可以逐帧混用。

| 帧的 `data` | 含义 |
|---|---|
| `raw:uint8:<base64>` | 彩色原始像素，字节数 = H×W×3 |
| `raw:uint16:<base64>` | 深度原始像素，字节数 = H×W×2 |
| 无 `raw:` 前缀 | 按 §3.1 的 `extra_info.image_format` 处理（默认 JPEG） |

彩色帧用了 `raw:uint16:`（或深度帧用了 `raw:uint8:`）、字节数与 `shape` 不符时，`/submit` 返回 500 和原因。

**(b) 二进制请求体**：`Content-Type: application/vnd.manifold.observation.v1`，必须带 `Content-Length`。

```text
5 bytes     magic = b'MIMG\x01'
4 bytes     JSON header 长度 L，uint32 big-endian
L bytes     UTF-8 JSON header：其余字段与 JSON 请求相同；每帧改为
            {"shape":[H,W,3],"format":"bgr8"} 或 {"shape":[H,W],"format":"depth16le"}，不含 data
剩余 bytes  像素字节，按 img_front → img_left → img_right → img_front_depth → img_left_depth → img_right_depth 拼接
```

- 限制：header ≤ 1 MiB，整个请求体 ≤ 64 MiB，每个维度 1..4096；三路彩色帧数相同，深度三路都为空或与彩色帧数相同；字节数与 `shape` 完全匹配，无多余尾部字节。
- 出错返回 **HTTP 400** 和 `{"status": "error", "message": "<原因>"}`，请求不进入推理。
- 比 raw Base64 JSON 少一次 base64，请求体小约 25%（3 路 640×480 约 2.8 MB）。

**`GET /info`**：在原有字段（`height`、`width`、`infer_count`、`request_obs_num`、`request_obs_interval`）之外增加 `"raw_image_protocol": 1`、`"binary_image_protocol": 1`，客户端可据此选择格式。

**与 UBT Agent RTC API 的关系**：图像格式（本节）和客户端动作前缀（§6.1）都已实现。在线的 RTC 端点 `/info` 返回 `rtc_prefix_protocol: 1`、`rtc_timeline_protocol: 1`、`rtc_max_prefix_steps: 10`，按该 API 实现的 Agent 可以直接接入。

## 4. 回复体（`GET /result/<ts>` → 200）

| 字段 | 类型 | 说明 |
|---|---|---|
| `timestamp` | float | 回显请求的 `timestamp` |
| `gen_action_rate` | int | 50 |
| `left_arm_actions` / `right_arm_actions` | list[50] | 每项 `{"joint_state": [8], "joint_velocity": [], "joint_effort": [], "end_pose": [], "is_euler": false}`；`joint_state` = 7 关节绝对目标（弧度）+ 1 夹爪，按 30 fps。带 `extra_info.rtc` 的请求：行 `[0:N]` 就是客户端发来的前缀（逐值相同），行 `[N:50]` 是新预测（§6.1）；否则行 0 对应请求那帧观测的时刻 |
| `ctrl_mode` | int | 2（关节位置控制） |
| `extra_info` | object | **可选**，见 §5。没有可回显的内容时整个键不出现 |

## 5. `extra_info`（回复）

键随端点不同，以后可能加新键。

### 5.1 键一览

| 键 | 类型 | 出现在 | 含义 |
|---|---|---|---|
| `rtc_timeline` / `rtc_prefix_steps` | string / int | 请求带 `extra_info.rtc` 时 | 回显 `"published_steps_v1"` 和本次前缀行数 N |
| `target_bbox` | 见右 | 带目标框叠加的端点 | 本次画在腕部图上、送进模型的框（像素 `[x1, y1, x2, y2]`），没画框为 `null`。单腕模型：一个框；双腕模型：长度 2 的列表 `[左腕框, 右腕框]`，每项可单独为 `null` |
| `yolo` | object | 配了腕部 YOLO 辅助模型的端点（当前两个在线端点都有） | 每条手臂：是否夹着箱子、夹爪与箱子的关系（hold / hit / other 及概率）、当前技能、技能内进度，见 §5.5 |
| `hold_prob` / `holding` | float / bool（双腕为长度 2 的列表） | 另配了夹持判别头的双腕端点 | 旧的夹持读数，只看左腕，右腕恒为 `null`。保留是为了兼容，新客户端请用 `yolo` |
| `skill` / `model_skill` / `phase_progress` / `skill_probs` | | 带技能头、且**没有** `yolo` 的端点（当前在线的都不发） | 策略模型的技能名、技能头估计、进度 0~1、各技能概率。端点有 `yolo` 时这四个键不出现 |

### 5.2 `target_bbox`

**坐标**：像素，基于对应腕部图（`img_left` / `img_right`）的**原始分辨率**（当前 640×480），不是模型输入的 224×224；两种图像格式下相同。原点左上角，x 向右、y 向下；整数，已裁剪到图像范围内。

**客户端自带框（优先，仅单腕模型）**：请求的 `extra_info.target_bbox` 若存在且合法，服务端直接画这个框，不跑检测。
- 格式：`[x1, x2, y1, y2]`，**比例**（0~1）。注意顺序是 x1, x2, y1, y2，与回复的像素 `[x1, y1, x2, y2]` 不同。
- 不合法（不满足 `0 ≤ x1 < x2 ≤ 1`、`0 ≤ y1 < y2 ≤ 1`）时退回检测。
- 双腕模型忽略请求里的框，两个腕部都用检测。

**服务端检测**（请求没带框时，每次请求、推理之前）：YOLO 微调检测器，置信度阈值 0.5。
- 双腕模型（不跟踪）：左右腕各自检测，每帧取面积最大的框，与训练标注方式一致。
- 单腕模型（跟踪）：没有目标时取面积最大的框；已有目标时在上一次目标中心 0.7 倍框长边范围内找，优先同一跟踪 ID，否则取中心最近的；找不到时下一次重新选；新回合清空。
- 画框：绿色 (0,255,0)，3 px，画在原始分辨率的腕部图上，然后才缩放进模型。

**为 `null`**：没有框达到 0.5，或目标丢失且这次没有可选的框；此时对应腕部图上也没画框。

### 5.3 示例

```json
"extra_info": {"target_bbox": [287, 142, 396, 319]}            // 单腕，画了框
"extra_info": {"target_bbox": null}                            // 单腕，没画框
"extra_info": {"target_bbox": [[287, 142, 396, 319], null]}    // 双腕，左腕有框、右腕没有
```

### 5.4 客户端读取约定

- 用 `rep.get("extra_info") or {}` 读取：缺失的键表示该端点不提供，不是出错。
- 忽略不认识的键。
- `target_bbox` 为 `null` 时不要沿用上一次的框当作模型看到的框。
- 双腕模型的 `target_bbox` 是列表的列表；可以看第一个元素是不是列表来区分。

### 5.5 `yolo`：夹持 / 接触 / 技能 / 进度（Oct 23 晚上线，Oct 24 加接触）

来源是一个独立的腕部 YOLO 模型（Oct 24 起为统一模型：检测器 + 技能 / 进度 / 接触 / 目标框几个小头），**不是**策略模型。每次请求、在画目标框之前，用该手臂腕部相机的原图和该手臂当前状态（7 关节 + 夹爪宽度 + 左右臂标记）各算一次；只看当前一帧，不带历史。

```json
"extra_info": {
  "yolo": {
    "left": {"hold": false, "hold_prob": 0.0, "skill": "focus", "skill_prob": 0.5321, "progress": 0.2747,
             "contact": "hit", "contact_prob": 0.958,
             "contact_probs": {"other": 0.042, "hold": 0.0, "hit": 0.958}}
  }
}
```

| 字段 | 类型 | 含义 |
|---|---|---|
| `left` / `right` | object | 端点控制哪几条手臂就有哪几个键：双臂模型两个都有，左臂模型只有 `left` |
| `hold` | bool | 是否夹着箱子，即 `hold_prob >= 0.5` |
| `hold_prob` | float 0~1 | 夹持概率 |
| `skill` | string | 概率最大的技能，取值见下 |
| `skill_prob` | float 0~1 | 该技能的概率（17 类 softmax） |
| `progress` | float 0~1 | 当前技能内的进度 |
| `contact` | string | 夹爪与箱子的关系，三类取概率最大的：`hold` = 夹着箱子；`hit` = 手指（张开或闭合的指尖）碰到/压到箱子但没夹住；`other` = 其他（没碰到箱子） |
| `contact_prob` | float 0~1 | `contact` 那一类的概率 |
| `contact_probs` | object | 三类各自的概率 `{"other", "hold", "hit"}`，三者相加为 1（四舍五入到 4 位） |

`skill` 的取值（17 类）：`global_observe`、`approach`、`focus`、`close_to`、`close_gripper`、`wait_close`、`lift`、`carry`、`open_gripper`、`wait_open`、`to_safe`、`wait_safe`、`return`、`wait`、`lift_recovery`、`hesitate`、`other`。

注意：
- 读数逐帧独立，会抖动；需要稳定状态时请在客户端做平滑或连续 N 帧确认。
- `contact` 三个键只在统一模型的端点出现（Oct 24 起两个左臂在线端点都有）；没有这三个键时说明端点还是旧模型，请按“键缺失 = 不提供”处理。
- `hold` / `hold_prob` 与 `contact_probs.hold` 来自同一个接触头，两者一致；新客户端建议直接读 `contact`。
- 离线评测（留出录制，原图 640×480）：接触三类平衡准确率 0.93，`hit` 精度 0.80 / 召回 0.80，`hold` 准确率 0.999；技能准确率 0.78，进度平均误差 0.20。
- 旧模型（Oct 23）的已知问题——空闲右臂读出 `hold: true`——在统一模型上尚未在双臂端点复测。
- 耗时：每次请求多约 10–15 ms。

## 6. 回合（episode）规则

满足任一条件，服务端就把这次请求当作**新回合**：清空目标框跟踪和 RTC 的上一段动作。

- `timestamp` 比上一次请求小。
- 距上一次请求超过 `reset_after_idle_s` 秒——**当前在线的模型都没设这条**，等待不会重置。

所以开始新回合的方法是让 `timestamp` 回退。

### 6.1 客户端前缀 RTC（推荐，Oct 22 晚上线）

延迟由客户端自己测：它最清楚观测采集、编码、网络和等待回复各花了多久，以及实际发布到了第几步。服务端只接收客户端发来的动作，**全部**作为条件，不做任何估计。

请求的 `extra_info.rtc`：

| 字段 | 内容 |
|---|---|
| `timeline` | 固定 `"published_steps_v1"` |
| `prefix_steps` | 整数 N，`1..rtc_max_prefix_steps`（当前 10，即模型训练时的最大前缀） |
| `left_actions` / `right_actions` | 各 `N × 8`，客户端已承诺执行的绝对关节目标（7 关节弧度 + 夹爪），顺序同 `joint_state` |

- 服务端把这 N 行（双臂模型用两臂，单臂模型用左臂）换算到模型空间后作为固定前缀（`inference_delay = N`），模型从第 N 行起续写。
- 回复行 `[0:N]` 与请求前缀**逐值相同**，行 `[N:50]` 是新预测；客户端从第 N 行接着执行。单臂模型的右臂：`[0:N]` 为客户端的右臂前缀，之后保持在其最后一行。
- 回复 `extra_info` 回显 `rtc_timeline`、`rtc_prefix_steps`。
- `extra_info.rtc` 不合法（`timeline` 不对、N 越界或不是整数、行数或维度不符、NaN/Inf）时，`/submit` 返回 HTTP 500 和原因，该请求不进入推理。JSON 与 `MIMG` 二进制请求体相同。
- 服务端不依赖 `timestamp` 的差值，也不再按自己的耗时丢行或另算前缀。

```mermaid
sequenceDiagram
    participant R as 机器人
    participant S as 策略服务
    Note over R: 自测延迟，选定已承诺的 N 行
    R->>S: submit(观测, extra_info.rtc = N 行)
    Note over R: 执行这 N 行
    S->>S: N 行作固定前缀，模型续写到 50 行
    S-->>R: result(行 0..N-1 = 前缀原值, 行 N..49 = 新预测)
    Note over R: 从第 N 行接着执行
```

### 6.2 服务端估计 RTC（旧客户端，不带 `extra_info.rtc` 时）

不改协议，客户端约定如下：

- 请求在途时，机器人**继续执行上一段动作**；新动作到达后，**丢掉覆盖「观测时刻 → 现在」这段时间的前几行**（行 0 = 观测时刻，30 fps），从之后的行接着执行。
- 服务端记住上一次回复的动作和它的 `timestamp`。新请求时算 `s = round((本次 timestamp − 上次 timestamp) × 30)`（机器人当时执行到旧动作的第几行）和 `d = ceil(延迟 × 30)`（最多 10 行），把旧动作的 `[s, s+d)` 行当作新动作的固定前缀，模型从那里续写，所以新动作的第 d 行起与机器人正在走的轨迹衔接。
- 因此 `timestamp` 必须是观测采集时间（秒）。旧动作覆盖不到 `[s, s+d)`（例如两次请求间隔超过约 1.3 s）时，本次不加前缀，等同普通推理。
- 新回合清空上一段动作，第一次请求不加前缀。

```mermaid
sequenceDiagram
    participant R as 机器人
    participant S as 策略服务
    R->>S: submit(timestamp = t_k, 观测)
    Note over R: 继续执行旧动作的 s…s+d 行
    S->>S: 旧动作 [s, s+d) 作固定前缀，模型续写 50 行
    S-->>R: result(50 行，行 0 = t_k)
    Note over R: 丢掉已过去的前 d 行左右，从衔接处接着执行
```

## 7. 示例客户端

```python
import base64, time, cv2, numpy as np, requests

BASE = "http://<gateway>/<端点>"   # 换模型只改这里
PIXEL = True                       # False：JPEG（默认格式）

def jpeg_frame(img_bgr):
    ok, jpg = cv2.imencode('.jpg', img_bgr, [int(cv2.IMWRITE_JPEG_QUALITY), 100])
    return {"data": base64.b64encode(jpg).decode(), "shape": list(img_bgr.shape)}

def pixel_frame(img_bgr):
    """相机原图，HxWx3 uint8 BGR。不缩放、不补边、不转颜色。"""
    img = np.ascontiguousarray(img_bgr)
    return {"data": base64.b64encode(img.tobytes()).decode(), "shape": list(img.shape)}

def request(fmt_pixel):
    frame = pixel_frame if fmt_pixel else jpeg_frame
    return {
        "timestamp": ts,                               # 观测采集时间（秒）
        "prompt": "",                                  # 服务端忽略
        "img_front": [frame(front_img)], "img_left": [frame(left_img)], "img_right": [frame(right_img)],
        "img_front_depth": [], "img_left_depth": [], "img_right_depth": [],
        "left_arm_joint_state": [left_joints_8],       # 7 关节弧度 + 1 夹爪
        "right_arm_joint_state": [right_joints_8],
        "left_arm_joint_velocity": [], "right_arm_joint_velocity": [],
        "left_arm_joint_effort": [], "right_arm_joint_effort": [],
        "left_end_pose": [], "right_end_pose": [],
        "is_euler": False,
        "extra_info": {"image_format": "pixel/binary"} if fmt_pixel else {},
    }

ts = time.time()
r = requests.post(f"{BASE}/submit", json=request(PIXEL))
if r.status_code == 500 and PIXEL:                     # 旧服务端或帧有误：改回 JPEG
    print(r.text); r = requests.post(f"{BASE}/submit", json=request(False))
while (r := requests.get(f"{BASE}/result/{ts}")).status_code != 200:
    time.sleep(0.05)
rep = r.json()
left = [a["joint_state"] for a in rep["left_arm_actions"]]   # 50 × 8，行 0 = 观测时刻，30 fps
box = (rep.get("extra_info") or {}).get("target_bbox")      # 单腕：[x1, y1, x2, y2] 或 None；双腕：[左, 右]
```
