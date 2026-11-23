# 腕部深度图输入（可选，uint8 毫米）

Nov 23, 2026

## 结论

- 新增**可选**输入 `img_left_depth`：左腕深度图，`uint8 [H, W]`，**单位毫米**，值 = `min(深度_mm, 255)`，`0` = 无深度。
  超过 255 mm 的一律记 `255`。
- 量程说明：需求原文写 "25.5 mm"。uint8 每单位 1 mm 时上限是 **255 mm（25.5 cm）**，与训练数据的编码一致（腕部相机指尖处深度中位约 141 mm，p90 约 226 mm）；25.5 mm 会把几乎所有像素都截到上限。
- 与 `img_left` **同分辨率、逐像素对齐**（同一成像器；当前 640×480）。客户端不缩放、不裁剪。
- **可选，向后兼容**：不带深度的请求与以前完全相同，旧客户端无需改动；不用深度的模型收到深度也只是忽略。
- 先看 `GET /info`：有 `depth8_protocol: 1` 才发深度（老端点会把 `depth8` 块当格式错误回 400）。
  `depth_prompt: true` 表示该模型训练时用了深度，会读它；`false` 或缺省 = 接受但不用。

## 客户端换算

```python
mm = depth_raw.astype(np.float32) * MM_PER_UNIT        # 传感器原始单位 -> 毫米（按相机标定）
depth8 = np.clip(np.rint(mm), 0, 255).astype(np.uint8)  # >255 mm 截为 255；无效像素保持 0
```

## 编码

### MIMG 二进制（云端 RTC_v2）

深度块排在**全部彩色块之后**，顺序固定 front、left、right（只排有帧的相机）。描述：

```json
{"image_order": ["img_left", "img_front", "img_right"],
 "img_left":  [{"shape": [480, 640, 3], "format": "jpeg", "nbytes": 12345}],
 "img_front": [{"shape": [480, 640, 3], "format": "jpeg", "nbytes": 23456}],
 "img_right": [{"shape": [480, 640, 3], "format": "jpeg", "nbytes": 11111}],
 "img_front_depth": [],
 "img_left_depth":  [{"shape": [480, 640], "format": "depth8"}],
 "img_right_depth": []}
```

```text
MIMG\x01 | JSON 长度 | JSON | left JPEG | front JPEG | right JPEG | left depth8 (H×W 字节)
```

| `format` | 块内容 | 用途 |
|---|---|---|
| `depth8`（新） | `uint8` 行主序，H×W 字节，毫米，截到 255 | 模型读取的腕部深度 |
| `depth16le`（旧） | 小端 `uint16`，2×H×W 字节，传感器单位 | 仍接受，模型不读 |

### JSON（`/submit` 的 JSON 请求体）

| 写法 | 说明 |
|---|---|
| `{"data": "raw:uint8:<base64(H×W 字节)>", "shape": [H, W]}` | 原始像素，前缀给出类型 |
| `{"data": "<base64(PNG)>", "shape": [H, W]}` | `extra_info.image_format` 缺省（`jpeg/base64`）时：8 位灰度 PNG（无损） |
| `{"data": "<base64(H×W 字节)>", "shape": [H, W], "dtype": "uint8"}` | `image_format: pixel/binary` 时；不写 `dtype` 按旧的 `uint16` |

- 深度按相机可选：每个 `img_*_depth` 列表要么为空，要么与彩色帧数相同；只发 `img_left_depth` 即可。
- 带宽：640×480 的 `depth8` 约 300 KB（三路 JPEG 合计约 170–200 KB），1 GbE 上传约多 2.5 ms；JSON 请用 PNG。

## 服务端如何使用

- 深度训练的模型（`/info` `depth_prompt: true`）：
  取本次查询左腕叠加层检测到的两个指尖，连线两端各外延 10 px，均匀取 10 个点；每点取 5×5 窗内有效（>0）像素的中位数，
  没有则取 20 px 内最近的有效像素；码 = `min(mm, 255)`。提示词末尾追加 `<prompt_sep>depth=d1 d2 ... d10`，
  与训练时写入的完全相同。
- 不加深度标签的情况（与训练里 20% 丢弃、无深度片段一致）：请求没有 `uint8` 的 `img_left_depth`；指尖少于 2 个；
  任一点没有有效深度。

## 错误（HTTP 400 / SDK 500，不进入推理）

`depth8` 用在彩色键上；深度 `shape` 不是 `[H, W]`；块字节数不符；深度帧数与彩色帧数不同；`raw:` 前缀不是
`uint8` / `uint16`；PNG 解不开、不是 8/16 位灰度或尺寸与 `shape` 不符；`dtype` 不是 `uint8` / `uint16`。
