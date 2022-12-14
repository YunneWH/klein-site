---
title: 配置
icon: config
order: 5
---

## [storage]

| 参数                 | 说明 |
| -------------------- | ---- |
| path                 |  数据存储位置    |
| max_summary_size     |  最大Summary日志大小，用于恢复数据库中的数据，默认：134217728    |
| max_level            |  lsm的最大层数，取值范围0-4，默认：4    |
| base_file_size       |  单个文件数据大小，默认：16777216    |
| compact_trigger      |  出发compation的文件数量, 默认：4    |
| max_compact_size     |  最大压缩大小，默认：2147483648    |
| dio_max_resident     |      |
| dio_max_non_resident |      |
| dio_page_len_scale   |      |

## [wal]

| 参数    | 说明 |
| ------- | ---- |
| enabled | 是否启用WAL，默认：false     |
| path    | 预写日志路径     |
| sync    | 同步写入WAL预写日志，默认false     |

## [cache]

| 参数                 | 说明 |
| -------------------- | ---- |
| max_buffer_size      |  最大缓存大小，默认：134217728    |
| max_immutable_number |  immemtable的最大值, 默认：4    |

## [log]

| 参数  | 说明 |
| ----- | ---- |
| level |  日志等级（debug、info、error、warn），默认：info   |
| path  |  日志存放位置    |