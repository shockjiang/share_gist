# 云端 RTC_v2（JPEG 原始字节）

Oct 30, 2026

## 结论

- 机器人只做一次 JPEG 编码，JPEG 原始字节直接放进 `MIMG` 二进制请求体，**不做 Base64，不走 JSON 图像**。
- 图像块顺序改为**左腕在前**：`img_left → img_front → img_right`，并在 JSON 头里用 `image_order` 声明。服务端收到左腕 JPEG 的最后一个字节就开始解码并跑左腕 YOLO（目标框、指尖）和该路 SigLIP，此时头部、右腕还在上传。
- 旧格式全部保留：原图 `bgr8` 的 `MIMG`、JSON（JPEG base64、`raw:` 前缀、`pixel/binary`）照常可用；不带 `image_order` 时按原顺序 `img_front → img_left → img_right` 解析。

## 客户端

前端通信方式选择“云端 RTC_v2（JPEG）”；YAML 模型配置使用 `communication: server_rtc_v2`。模型 URL 仍可独立配置。串行/并行调度、30 Hz、50 帧回包、RTC 前缀及时间戳校验均沿用现有云端 RTC。

每路相机在收到图像后解码为 BGR，预先编码 JPEG（质量 100，不缩放），将像素、JPEG 和原始传感器时间戳一起放入最多三帧的缓存。独立相机回调线程完成转换，不占用动作处理回调线程。取观测时按原有时间窗对齐三路相机与关节状态，直接引用选中帧的 JPEG；提交时只封装，不重复压缩。编码失败的帧不入缓存；没有新鲜对齐帧则按原有超时处理。JPEG 为有损编码，质量 100 仍不保证逐像素相同。

**封装顺序：左腕 JPEG 第一个写入请求体**，然后头部、右腕；JSON 头带 `"image_order": ["img_left", "img_front", "img_right"]`。发送端应边封装边发送（或至少保证左腕块在请求体最前），左腕块越早到，服务端越早开始检测。

## MIMG 线格式

HTTP `POST /submit`，`Content-Type` 仍为 `application/vnd.manifold.observation.v1`，必须带 `Content-Length`。

```text
MIMG\x01 | JSON 字节数（4 字节，大端） | UTF-8 JSON | left JPEG | front JPEG | right JPEG
```

| 字段 | 字节数 | 内容 |
|---|---|---|
| Magic | 5 | 十六进制 `4d 49 4d 47 01`，即 `b'MIMG\x01'` |
| JSON 长度 | 4 | 无符号大端整数，按 UTF-8 字节数计算 |
| JSON 头 | 上一字段给出的长度 | 完整观测元数据 |
| 图像块 1..3 | 各块的 `nbytes` | 按 `image_order` 的顺序：左腕、头部、右腕 JPEG 原始字节 |

三路彩色图片各一个描述，例如：

```json
{"image_order": ["img_left", "img_front", "img_right"],
 "img_left":  [{"shape": [480, 640, 3], "format": "jpeg", "nbytes": 12345}],
 "img_front": [{"shape": [480, 640, 3], "format": "jpeg", "nbytes": 23456}],
 "img_right": [{"shape": [480, 640, 3], "format": "jpeg", "nbytes": 11111}]}
```

- `image_order`：可选；三个彩色相机名各出现一次。不带时为 `["img_front", "img_left", "img_right"]`（旧顺序）。只决定图像块在请求体里的先后，与 JSON 里键的先后无关。
- `shape`：解码后的 BGR uint8 HWC 形状，H/W 均为 1..4096；JPEG 解码结果必须正好是这个形状。
- `format`：`"jpeg"`（本协议）或 `"bgr8"`（原图，字节数 = H×W×3）；同一请求里可以混用。
- `nbytes`：该 JPEG 块的精确字节数（`format: "jpeg"` 时必填）。
- 图片块为 JPEG 原始字节，不做 Base64。深度数组为空（若发深度，`depth16le` 块排在全部彩色块之后，顺序固定为 front、left、right）。其他观测和 RTC 字段不变。
- 限制：JSON 头 ≤ 1 MiB；整个请求 ≤ 64 MiB；解码后图片合计 ≤ 64 MiB；包末尾不允许额外字节。

JSON 头其余字段（数组外层均为当前单帧，不是历史窗口）：

| 字段 | 结构 | 含义 |
|---|---|---|
| `timestamp` | 正数 double | 头相机原始 Unix 秒时间戳，会话内严格递增 |
| `prompt` | string | 任务提示词（服务端使用训练时的固定提示词） |
| `is_euler` | false | 末端位姿使用四元数 |
| `left/right_arm_joint_state` | [[8 个数]] | 七个臂关节位置 + 夹爪开口 |
| `left/right_arm_joint_velocity` | [[8 个数]] | 对应速度，使用反馈数据单位 |
| `left/right_arm_joint_effort` | [[8 个数]] | 对应 effort，使用反馈数据单位；夹爪项来自电流反馈 |
| `left/right_end_pose` | [[x,y,z,qx,qy,qz,qw,gripper]] | torso_link 下腕部位姿，位置及夹爪为 m |
| `waist_joint_state/velocity/effort` | [[1 个数]] | waist_yaw_joint |
| `leg_joint_state/velocity/effort` | [[3 个数]] | lifter_pitch_1_joint、lifter_pitch_2_joint、lifter_pitch_3_joint |
| `img_front/img_left/img_right` | [{shape,format,nbytes}] | 每路一个 JPEG 描述，无 data 字段 |
| `img_front_depth/img_left_depth/img_right_depth` | [] | 当前不传深度 |
| `image_order` | [3 个相机名] | 图像块顺序，RTC_v2 固定发 `["img_left","img_front","img_right"]` |
| `extra_info.rtc` | object | RTC 承诺前缀，结构如下 |

左右臂关节顺序分别为 `L_` / `R_` 前缀的：shoulder_pitch_joint、shoulder_roll_joint、shoulder_yaw_joint、elbow_roll_joint、elbow_yaw_joint、wrist_pitch_joint、wrist_roll_joint。位置单位为 rad，第八项夹爪开口单位为 m。

`extra_info.rtc` 包含：

- `timeline`：固定 `published_steps_v1`。
- `prefix_steps`：N，整数 1..10。
- `left_actions` / `right_actions`：各 N 个八维位置数组，不是带 `joint_state` 字段的对象。

## 能力协商（`GET /info`）

```json
{"request_obs_num": 1, "request_obs_interval": 0, "width": 640, "height": 480,
 "rtc_prefix_protocol": 1, "rtc_timeline_protocol": 1,
 "binary_image_protocol": 1, "binary_jpeg_protocol": 1}
```

- `binary_jpeg_protocol: 1` 表示服务端接受 `format: "jpeg"` 图像块和 `image_order`。两项能力是一起加入的，所以有 `binary_jpeg_protocol` 就可以按左腕优先的顺序发。
- 缺少 JPEG 能力时客户端报错，不回退为原图或 JSON，也不重复提交失败的请求。旧 `rtc` / `server_rtc` 仍使用原有能力协商与原图格式。
- `width`、`height` 为模型声明的输入尺寸；本传输层不缩放，发相机原图（640×480）即可，缩放由服务端按训练方式完成。

## 提交与领取

```bash
curl --fail "$MODEL_BASE_URL/info"
curl --fail -X POST "$MODEL_BASE_URL/submit" \
  -H 'Content-Type: application/vnd.manifold.observation.v1' \
  --data-binary @observation.mimg
curl --fail "$MODEL_BASE_URL/result/$TIMESTAMP_KEY"
```

- 正常入队：`{"status":"success","message":"任务已接收，正在处理"}`。
- 格式或 JPEG 校验失败（`image_order` 不合法、`nbytes` 缺失或越界、JPEG 解不开或尺寸与 `shape` 不符、包被截断或有尾部字节）：HTTP 400，`{"status":"error","message":"具体原因"}`，请求不进入推理。
- 客户端在提交前检查观测新鲜度：`0 <= 当前时间 - timestamp < 50/30` 秒。提交成功只表示入队；同一模型会话不并发提交。提交失败或超时不得自动重发，也不回退到原图/JSON。
- `TIMESTAMP_KEY` 为提交的原始 double 时间戳的 Python `str(float(timestamp))` 形式。未完成返回 202 `{"status":"processing","message":"任务正在处理中"}`；完成返回 200，结构见下。
- 只有明确收到 202 时才继续轮询（约 10 ms 间隔，受整轮推理超时限制）；成功领取会从服务端队列移除结果，只能领取一次；GET 超时不能当作 202 重试。

| 回复字段 | 约束 |
|---|---|
| `timestamp` | 数值与提交时完全一致 |
| `ctrl_mode` | 固定 2，关节控制 |
| `gen_action_rate` | 30 |
| `left_arm_actions` / `right_arm_actions` | 各恰好 50 个 `{"joint_state":[8 个数]}` |
| `waist_actions` / `leg_actions` | 可省略或为空 |
| `extra_info.rtc_timeline` | 固定 `published_steps_v1` |
| `extra_info.rtc_prefix_steps` | 等于本次请求的 N |

双臂前 N 帧与请求的承诺前缀逐值绝对误差不超过 1e-5；所有数值有限。

## 服务端

- 我方策略服务已内置本协议：解析 `jpeg` 块与 `image_order`，每块 JPEG 在字节到齐时解码一次（BGR），解码结果直接交给逐相机提前启动的检测与视觉编码，整包校验通过后再进入推理队列。
- 第三方 SDK 服务若要支持本协议，接入的适配器同样需要支持 `jpeg` 块与 `image_order`。
