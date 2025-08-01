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

---
