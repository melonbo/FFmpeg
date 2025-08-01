# **av_read_frame**
`av_read_frame`是FFmpeg libavformat库中的核心解复用函数，用于从已打开的媒体文件中读取数据包。 该函数位于`libavformat/demux.c`文件中，是解复用过程的主要接口。
### 检查时间戳

函数首先检查是否启用了时间戳生成标志(`AVFMT_FLAG_GENPTS`)，​当输入流中没有时间戳（或时间戳不完整）时，由 FFmpeg 自动生成合理的 **PTS（Presentation Time Stamp）**。

1. **非时间戳生成模式**：如果未启用时间戳生成，函数会优先从包缓冲区读取数据包，否则调用`read_frame_internal`读取新数据包。
2. **时间戳生成模式**：当启用时间戳生成时，函数进入复杂的循环处理逻辑，处理包缓冲区中的数据包并进行时间戳修正。

常见场景：
- 原始流（如裸 H.264 数据）缺少时间戳。
- 网络流或实时流因丢包导致时间戳不连续。

 设置方法​：
 在打开输入流时​，通过 `AVFormatContext` 的 `flags` 字段设置：
 ```
 AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, input_file, NULL, NULL);

// 设置 AVFMT_FLAG_GENPTS
fmt_ctx->flags |= AVFMT_FLAG_GENPTS;

// 后续操作（如 avformat_find_stream_info）
```

### 时间戳处理

在时间戳生成模式下，函数会：
- 检查包缓冲区中是否有待处理的数据包
- 对于缺失PTS的数据包，尝试从后续数据包的DTS推导PTS值
- 处理文件结束时的特殊情况，为最后的参考帧设置PTS

### 数据包后处理

函数在返回数据包前会进行最终处理：

- 为关键帧添加索引条目
```
if ((s->iformat->flags & AVFMT_GENERIC_INDEX) && pkt->flags & AV_PKT_FLAG_KEY) {
        ff_reduce_index(s, st->index);
        av_add_index_entry(st, pkt->pos, pkt->dts, 0, 0, AVINDEX_KEYFRAME);
    }
```
- 调整相对时间戳
```
    if (is_relative(pkt->dts))
        pkt->dts -= RELATIVE_TS_BASE;
    if (is_relative(pkt->pts))
        pkt->pts -= RELATIVE_TS_BASE;
```


`av_read_frame`主要依赖`read_frame_internal`函数来实际读取数据包。  
`read_frame_internal`函数负责处理解析器初始化、数据包解析和时间戳验证等底层操作。

`av_read_frame()` 是 FFmpeg 中用于 ​**从媒体文件中读取数据包（`AVPacket`）的核心函数**，属于解复用（Demux）阶段的关键操作。以下从 ​**功能原理、工作流程、内存管理、异常处理**​ 等角度进行深度解析：

---

### 一、函数定义与基本作用

#### 1. 函数原型

```
int av_read_frame(AVFormatContext *s, AVPacket *pkt);
```

- ​**参数**​：
    - `s`：`AVFormatContext` 对象，管理输入源（文件/网络流）的上下文。
    - `pkt`：输出的数据包，需预先初始化（通常通过 `av_init_packet()`）。
- ​**返回值**​：
    - `0`：成功读取一个数据包。
    - `AVERROR_EOF`：到达文件末尾。
    - 其他负数：错误码（如 I/O 错误、数据损坏）。

#### 2. 核心功能

- ​**读取一个完整的压缩数据包**​（视频帧/音频帧/字幕等）。
- ​**处理时间戳**​（PTS/DTS）和流索引映射。
- ​**自动管理数据包内存**​（内部分配 `pkt->data` 缓冲区）。


### 二、底层工作流程

`av_read_frame()` 的实际实现通过调用内部函数 `read_frame_internal()` 完成，其流程如下：

#### 1. 初始化检查

```
if (!pkt->data) 
    av_init_packet(pkt);  // 确保 pkt 已初始化
else if (pkt->buf) 
    av_packet_unref(pkt); // 避免内存泄漏
```

#### 2. 调用解复用器的 `read_packet` 方法

- 根据输入格式（MP4/FLV/TS等）调用对应的解复用器：
    
    ```
    ret = s->iformat->read_packet(s, pkt);  // 如 ff_mp4_read_packet()
    ```
    
    - ​**MP4 示例**​：解析 `moov`/`mdat` 盒子，提取 H.264/H.265 数据包。
    - ​**TS 流示例**​：处理 PAT/PMT 表，定位音视频 PID。

#### 3. 时间戳处理

- ​**修正无效时间戳**​（若启用 `AVFMT_FLAG_GENPTS`）：
    
    ```
    if ((s->flags & AVFMT_FLAG_GENPTS) && pkt->pts == AV_NOPTS_VALUE)
        pkt->pts = pkt->dts;  // 使用 DTS 作为后备
    ```
    
- ​**时间基转换**​（从流时间基到全局时间基）：
    
    ```
    av_packet_rescale_ts(pkt, st->time_base, AV_TIME_BASE_Q);
    ```
    

#### 4. 流索引校验

确保 `pkt->stream_index` 有效：

```
if (pkt->stream_index >= s->nb_streams) {
    av_packet_unref(pkt);
    return AVERROR_INVALIDDATA;
}
```

### 三、内存管理机制

#### 1. 数据包内存分配

- ​**`pkt->data` 缓冲区**​：由 FFmpeg 内部通过 `av_malloc()` 分配。
- ​**内存所有权**​：调用者必须通过 `av_packet_unref()` 释放内存。

#### 2. 典型内存问题与修复

|​**问题**​|​**原因**​|​**修复方法**​|
|---|---|---|
|内存泄漏|未调用 `av_packet_unref()`|每次循环结束前释放包|
|双重释放|重复调用 `av_packet_unref()`|释放后置 `pkt->data = NULL`|
|野指针访问|释放后继续使用 `pkt->data`|遵循“分配-使用-释放”生命周期|

---
# read_frame_internal()

### ​**1. 函数功能**​

- ​**核心任务**​：  
    从 `AVFormatContext` 对应的输入源中读取一个完整的数据包（`AVPacket`），处理时间戳、边界校验、流索引映射等。
- ​**调用路径**​：  
    `av_read_frame()` → `read_frame_internal()`
- ​**关键输出**​：  
    填充有效的 `AVPacket`，包含：
    - 压缩数据（`data` 和 `size`）
    - 时间戳（`pts`、`dts`、`duration`）
    - 流索引（`stream_index`）

### ​**2. 函数原型与参数**​

```
static int read_frame_internal(AVFormatContext *s, AVPacket *pkt);
```

- ​**参数**​：
    - `s`：`AVFormatContext`，管理输入源的上下文。
    - `pkt`：输出的数据包，需已通过 `av_init_packet()` 初始化。
- ​**返回值**​：
    - `0`：成功读取一个包。
    - `AVERROR_EOF`：到达文件尾。
    - 其他负数：错误码（如网络中断、数据损坏）。

### ​**3. 核心流程**​

#### ​**​(1) 初始化检查**​

```
if (!pkt->data)  // 检查 pkt 是否已初始化
    av_init_packet(pkt);
else if (pkt->buf)  // 避免重复使用未释放的 pkt
    av_packet_unref(pkt);
```

#### ​**​(2) 从底层读取数据**​

- ​**调用解复用器的 `read_packet` 方法**​（如 `ff_mp4_read_packet` 对于 MP4 文件）：
    
    ```
    ret = s->iformat->read_packet(s, pkt);
    ```
    
    - 不同封装格式（MP4、FLV、TS等）实现各自的 `read_packet` 逻辑。

#### ​**​(3) 时间戳处理**​

- ​**修正无效时间戳**​（若启用 `AVFMT_FLAG_GENPTS`）：
    
    ```
    if ((s->flags & AVFMT_FLAG_GENPTS) && pkt->pts == AV_NOPTS_VALUE)
        pkt->pts = guess_correct_pts(s, pkt->dts);
    ```
    
- ​**时间基转换**​（从流时间基到全局时间基）：
    
    ```
    av_packet_rescale_ts(pkt, st->time_base, AV_TIME_BASE_Q);
    ```
    

#### ​**​(4) 流索引校验**​

- 确保 `pkt->stream_index` 在有效范围内：
    
    ```
    if (pkt->stream_index >= s->nb_streams) {
        av_packet_unref(pkt);
        return AVERROR_INVALIDDATA;
    }
    ```
    

#### ​**​(5) 边界与错误处理**​

- ​**处理不完整数据包**​（如网络流分片）：
    
    ```
    if (pkt->flags & AV_PKT_FLAG_CORRUPT)
        av_log(s, AV_LOG_WARNING, "Corrupt packet detected\n");
    ```
    
- ​**处理 EOF**​：
    
    ```
    if (ret == AVERROR_EOF)
        s->pb->eof_reached = 1;
    ```

---
# av_packet_make_refcounted()

**把一份“裸数据”的 AVPacket 变成“带引用计数”的 AVPacket——如果它还没有引用计数，就给它建一个；如果已经有了，就直接返回成功。**

`pkt->buf != NULL`	
数据已经引用计数 → 什么都不做，直接返回 0	
`pkt->buf == NULL`	
新建 AVBufferRef：
1. 申请大小为 `pkt->size + AV_INPUT_BUFFER_PADDING_SIZE` 的 buffer
2. 把 `pkt->data` 拷贝到新 buffer
3. 把 `pkt->data` 指向新 buffer
4. 引用计数置 1	
