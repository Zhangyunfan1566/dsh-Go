# 图片 / 文件上传通道：字节到底在哪一层被改写

本文只回答一件事：**从手机选一张图到它落到 Host，字节在哪一步被重新编码，为什么带文本块（tEXt / iTXt / zTXt / EXIF）的 PNG 会被压成 JPEG 或 WebP。**

结论先行：

1. **App 相册与拍照两条路，在选图那一刻就已经被重编码了**（`image_picker` 的 `imageQuality: 88`）。这一层丢块、丢 EXIF，且和 Host 无关 —— 换任何 Host 都救不回来。
2. **App 的「文件」入口不重编码**，字节原样进 `POST /api/session/uploadFileBinary`，且**不带 `mediaType` 查询参数**时 Host 逐字节落盘。
3. **Host 的附件归一化（`dsh-attachment-local`）只在超限时才重编码**。判定函数 `canPassThroughNormalization` **不看 metadata**，所以「带文本块」本身不是触发条件；触发条件是像素预算 / 长边 / 字节数三项之一。
4. 一旦归一化真的动手，**PNG 一律变成 JPEG（有 alpha 则 WebP）**，文本块与 EXIF 全部丢弃。这是设计如此，不是 bug。

---

## 1. App 侧：三个入口，两种命运

`app/lib/ui/chat/widgets/plus_panel.dart:48` 定义入口：

```dart
enum AttachSource { gallery, camera, files }
```

分派在 `app/lib/ui/chat/chat_screen.dart:1092`：

```dart
picked = switch (source) {
  AttachSource.gallery => PickedAttachments(await picker.pickImages(limit: room)),
  AttachSource.camera  => PickedAttachments([?await picker.takePhoto()]),
  AttachSource.files   => await picker.pickFiles(limit: room, heldBytes: totalAttachmentBytes(_drafts)),
};
```

### 1.1 相册 / 拍照 —— 第一处重编码（不可绕过）

`app/lib/state/attachments.dart:141-173`：

```dart
class DeviceAttachmentPicker implements AttachmentPicker {
  // Phone photos are 12–50 MP; the model gains nothing past ~2.5k on the long
  // side and the upload shrinks tenfold. Re-encoding also turns HEIC into JPEG.
  static const _maxSide = 2560.0;
  static const _quality = 88;

  @override
  Future<List<DraftAttachment>> pickImages({required int limit}) async {
    final picked = await ImagePicker().pickMultiImage(
      maxWidth: _maxSide,
      maxHeight: _maxSide,
      imageQuality: _quality,
      limit: limit,
      requestFullMetadata: false,
    );
    return [for (final x in picked.take(limit)) DraftAttachment.sniff(x.name, await x.readAsBytes())];
  }
```

`image_picker` 在 Android 上，只要 `imageQuality != null` 或 `maxWidth/maxHeight != null`，就走 `ImageResizer` 解码 → 缩放 → `Bitmap.compress(JPEG, quality)` 重新编码，并写进缓存目录再返回。**返回值已经不是原文件**：

- PNG → JPEG（Android 侧压缩格式固定为 JPEG，`imageQuality` 只调质量）
- 透明通道被拍平
- tEXt / iTXt / zTXt / EXIF / ICC 全部丢失
- 文件名变成 `image_picker_xxx.jpg`

`requestFullMetadata: false` 只影响**读取** EXIF 的权限，不影响**写出**时丢元数据。

> 这一层在 1.0.0 首版就存在：`git log -S "imageQuality: _quality"` 只命中 `a5fa6738 DSH Go 1.0.0`。

### 1.2 文件入口 —— 不重编码

`app/lib/state/attachments.dart:176-197`：

```dart
Future<PickedAttachments> pickFiles({int limit = maxAttachments, int heldBytes = 0}) async => _readAttachmentFiles(
  (await FilePicker.pickFiles()).map(_AttachmentFileInput.platform),
  limit: limit,
  heldBytes: heldBytes,
);
```

`_readAttachmentFiles`（同文件 199-274 行）**只做签名嗅探与预算检查，从不重编码**：

```dart
await for (final chunk in file.readAsByteStream()) {
  final take = (prefix.length - prefixLength).clamp(0, chunk.length);
  prefix.setRange(prefixLength, prefixLength + take, chunk);
  prefixLength += take;
  mediaType = inlineImageType(Uint8List.sublistView(prefix, 0, prefixLength));
  final budget = mediaType == null ? maxFileBytes : maxImageBytes;
  ...
  bytes.add(chunk);   // 原样保留
}
```

`inlineImageType`（同文件 72-90 行）按魔数判定，PNG / JPEG / GIF / WebP 四选一：

```dart
if (at(0, const [0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A])) return 'image/png';
if (at(0, const [0xFF, 0xD8, 0xFF])) return 'image/jpeg';
if (at(0, const [0x47, 0x49, 0x46, 0x38])) return 'image/gif';
if (at(0, const [0x52, 0x49, 0x46, 0x46]) && at(8, const [0x57, 0x45, 0x42, 0x50])) return 'image/webp';
```

**所以：只要走「文件」入口选同一张 PNG，字节就是原样的。**

### 1.3 发送：图片内联，文件走上传

`app/lib/state/chat_state.dart:1547-1597`：

```dart
final images = [
  for (final a in attachments)
    if (a.isImage) PromptImage.fromBytes(a.bytes, mediaType: a.mediaType!, name: a.name),
];
...
await prompt(images, staged);
```

`PromptImage.fromBytes`（`packages/dsh_client/lib/src/session_api.dart:38`）只是 base64：

```dart
factory PromptImage.fromBytes(List<int> bytes, {required String mediaType, String? name}) =>
    PromptImage(mediaType: mediaType, base64Data: base64Encode(bytes), name: name);
```

图片走 `session/prompt` 的 `content[].type == 'image'`（内联 base64），非图片走 `_stageFiles` → `uploadFile`。

---

## 2. Host 侧：文件上传通道

`packages/dsh_client/lib/src/session_api.dart:494-547`：

```dart
Future<FileReceipt> uploadFile(String sessionId, List<int> bytes, {String? name}) async {
  final streamed = await _uploadFileBinary(sessionId, bytes, name: name);
  if (streamed != null) return streamed;
  if (bytes.length > _maxInlineUploadBytes) {   // 30 MiB
    throw const DshProtocolException(
      'Host has no streaming file upload route; a file this large cannot be staged inline',
    );
  }
  final value = asMapOrEmpty(
    await rpc.call('fileUploads/upload', {
      'agentId': sessionId,
      'request': {'data': base64Encode(bytes), 'name': ?name},
    }),
  );
  return FileReceipt(asString(value['receiptId']) ?? '', FileAttachment.fromJson(value['file']));
}
```

首选路由 `POST /api/session/uploadFileBinary`，404 时退回 `fileUploads/upload`（base64 单帧，上限 30 MiB）。

### 2.1 引擎侧：`mediaType` 参数是分水岭

`@deepseek-ai/dsh-client-file-upload/lib/index.js`（`handleFileUploadHttp`）：

```js
const name = url.searchParams.get("name") ?? void 0;
const imageMediaType = url.searchParams.get("mediaType") ?? void 0;
...
result = {
  ok: true,
  value: imageMediaType === void 0
    ? await service.uploadStream(upload)              // 逐字节落盘，不碰图片
    : await service.uploadImageStream({ ...upload, mediaType: imageMediaType })  // 走图片归一化
};
```

**不带 `mediaType` → `uploadStream`，字节原样。**
**带 `mediaType` → `uploadImageStream`，进入图片归一化，会被转码。**

App 的 `_uploadFileBinary` 只传 `sessionId` 和 `name`，不传 `mediaType`：

```dart
final response = await carrier.connection.postBytes(
  _fileUploadBinaryPath,
  bytes,
  query: {'sessionId': sessionId, 'name': ?name},
  ...
);
```

所以 App 的文件通道是**纯二进制直传**。

---

## 3. Host 侧：图片归一化（真正把 PNG 变 JPEG 的地方）

包：`@deepseek-ai/dsh-attachment-local`（本机 `0.2.2-rc.1`，sharp `0.35.5` / libvips `8.18.7`）。

### 3.1 直通判定

`lib/index.js:212`：

```js
function canPassThroughNormalization(detected, bytes, policy) {
  return detected.mediaType !== "image/gif"
    && !detected.animated
    && detected.depth === "uchar"
    && detected.space === "srgb"
    && bytes <= policy.maxBytes
    && detected.width * detected.height <= policy.maxPixels
    && Math.max(detected.width, detected.height) <= policy.maxDimension;
}
```

**注意这里没有 `carriesMetadata`。** 也就是说「图片带文本块」**不是**触发条件 —— 带 tEXt 的小 PNG 照样直通、字节不变。

### 3.2 默认预算

`lib/index.js:919-923`：

```js
const DEFAULT_NORMALIZED_IMAGE_MAX_PIXELS = 2048 * 2048;   // 4,194,304
const DEFAULT_NORMALIZED_IMAGE_MAX_DIMENSION = 8192;
const DEFAULT_NORMALIZED_IMAGE_MAX_BYTES = 4 * 1024 * 1024; // 4 MiB
```

`normalizeImage`（`lib/index.js:255`）：

```js
async function normalizeImage(data, detected, policy) {
  if (canPassThroughNormalization(detected, data.byteLength, policy)) return {
    data, mediaType: detected.mediaType, width: detected.width, height: detected.height
  };
  const sharp = requireSharp();
  const { width, height } = initialDimensions(detected, policy);
  const encoded = await encodeFirstWithinLimit(
    encodingLadder(preparedPipeline(sharp, data, width, height), detected.hasAlpha),
    policy.maxBytes
  );
  return await verifyNormalizedImage(...);
}
```

### 3.3 编码格式是硬编码的

`lib/index.js:84-87`：

```js
function encodingLadder(prepared, hasAlpha) {
  const mediaType = hasAlpha ? "image/webp" : "image/jpeg";
  return IMAGE_ENCODING_QUALITIES.map((quality) => (() => encode(prepared.clone(), mediaType, quality)));
}
```

质量阶梯 `[85, 75, 60]`（`lib/index.js:60-64`）。

**所以：一旦进归一化，PNG 必然变成 JPEG；有 alpha 则必然变成 WebP。没有「保持 PNG」这个选项。**
`preparedPipeline` 还会 `.rotate()`（按 EXIF 转正）并 `.toColourspace("srgb")`，元数据不显式保留 → 丢弃。

---

## 4. 实测

环境：本机 `@deepseek-ai/dsh` 内置 `dsh-attachment-local@0.2.2-rc.1` + `sharp@0.35.5`。
方法：直接 `import` 该包的 `canPassThroughNormalization` / `normalizeImage`，喂构造好的 PNG。

| 输入 | 尺寸 | 字节 | 直通 | 输出 |
|---|---|---|---|---|
| `plain.png` | 64×64 | 237 | ✅ | `image/png` 237 B，**字节完全相同** |
| `text.png`（注入 tEXt） | 64×64 | 273 | ✅ | `image/png` 273 B，**字节完全相同** |
| `withMeta`（带 EXIF） | 64×64 | 770 | ✅ | `image/png` 770 B，**字节完全相同** |
| `big_plain.png` | 3000×3000 | 125,128 | ❌ | `image/jpeg` 2048×2048 24,844 B |
| `big_text.png`（9 MP + tEXt） | 3000×3000 | 127,183 | ❌ | `image/jpeg` 2048×2048 24,844 B |
| `big_alpha.png` | 3000×3000 | 183,811 | ❌ | **`image/webp`** 2048×2048 11,362 B |
| 9 MP + EXIF | 3000×3000 | — | ❌ | `image/jpeg`，**输出 EXIF 已消失** |

关键读数：

- 64×64 带 tEXt / EXIF 的 PNG：`passThrough = true`，输出与输入**逐字节相同** → 文本块**不是**触发条件。
- 3000×3000（9 MP > 4.19 MP 预算）：`passThrough = false`，PNG → JPEG，**tEXt 与 EXIF 全丢**。
- 3000×3000 带 alpha：PNG → **WebP**。
- 9 MP + EXIF 转码后 `metadata().exif === undefined` → 元数据确实被丢弃。

---

## 5. 所以「带文本块的 PNG 被强转」到底是怎么回事

按可能性排序：

1. **图片超过 4.19 MP 或 4 MiB** —— 最常见。手机截图 / 立绘基本都超。此时 Host 归一化必然转 JPEG/WebP，文本块丢。**这是唯一由 Host 决定的路径。**
2. **从相册或拍照入口选的** —— 在 App 内就已经被 `imageQuality: 88` 转成 JPEG 了，Host 收到时压根不是 PNG。**此时无论 PNG 多小都救不回来。**
3. **带 alpha 的 PNG** —— 转码目标固定为 WebP，不是 JPEG。

如果一张 PNG **小于 4.19 MP、小于 4 MiB、8-bit sRGB、单帧、从「文件」入口选**，那么它会被逐字节原样保存，文本块完好。

---

## 6. 想保住原始 PNG，有哪些路

按代价从低到高：

1. **换成「文件」入口选图**（`＋` → 文件），而不是相册。这一条就能绕开 App 侧 `imageQuality: 88`，字节原样进 Host。
2. **控制尺寸**：长边 ≤ 2048 且总像素 ≤ 4,194,304（例如 2048×2048、2560×1600），文件 ≤ 4 MiB。这样 `canPassThroughNormalization` 直接放行，Host 一个字节都不动。
3. **不要带 alpha**：带 alpha 的源即便转码也只会去 WebP，不会回 PNG。
4. **调高 Host 预算**：`LocalAttachmentStore.Config` 暴露了 `normalizedImageMaxPixels` / `normalizedImageMaxDimension` / `normalizedImageMaxBytes`（`lib/index.js:985-987`）。调大它们能让更大的图直通。但这是 Host 部署侧配置，App 改不了；且调大后模型请求侧仍可能按路由 target 再缩一次。
5. **改代码**（需要 fork 并自建 APK）：
   - App 侧：把 `DeviceAttachmentPicker.pickImages` 的 `imageQuality` / `maxWidth` / `maxHeight` 去掉，改为走 `FilePicker`（`FileType.image`）读原始字节。这是唯一能保住「相册原图」的改法。
   - Host 侧：在 `encodingLadder` 里为 `image/png` 增加 PNG 候选（`sharp(...).png({compressionLevel: 9})`），或让 `canPassThroughNormalization` 对「带需要保留的元数据」的源放宽。注意 `verifyNormalizedImage` 会校验 `carriesMetadata === false`，改这里要一并处理。

---

## 7. 代码坐标速查

| 环节 | 位置 |
|---|---|
| 入口枚举 | `app/lib/ui/chat/widgets/plus_panel.dart:48` |
| 入口分派 | `app/lib/ui/chat/chat_screen.dart:1092-1096` |
| **相册重编码** | `app/lib/state/attachments.dart:141-173`（`_maxSide=2560`, `_quality=88`） |
| 文件读取（不重编码） | `app/lib/state/attachments.dart:199-274` |
| 魔数嗅探 | `app/lib/state/attachments.dart:72-90` |
| 图片内联 base64 | `packages/dsh_client/lib/src/session_api.dart:35-51` |
| 发送分派 | `app/lib/state/chat_state.dart:1547-1597` |
| 文件上传客户端 | `packages/dsh_client/lib/src/session_api.dart:494-547` |
| 上传路由（引擎） | `@deepseek-ai/dsh-client-file-upload/lib/index.js` `handleFileUploadHttp` |
| **直通判定** | `@deepseek-ai/dsh-attachment-local/lib/index.js:212` |
| **归一化主体** | `@deepseek-ai/dsh-attachment-local/lib/index.js:255` |
| **编码格式硬编码** | `@deepseek-ai/dsh-attachment-local/lib/index.js:84-87` |
| 默认预算 | `@deepseek-ai/dsh-attachment-local/lib/index.js:919-923` |
| 配置 schema | `@deepseek-ai/dsh-attachment-local/lib/index.js:978-988` |
| 鸿蒙侧归一化补丁 | `runtime/patches/ohos-image-codec.diff`（仅 `process.platform === "openharmony"`） |

## 8. 顺带说明：`image_shrink.dart` 不是这条链路

`app/lib/share_code/image_shrink.dart` 看起来很像元凶（长边 1600、JPEG q82、透明走 PNG），但它只服务**会话分享导出**（`conversation_share.dart`），和聊天上传无关。别改错地方。
