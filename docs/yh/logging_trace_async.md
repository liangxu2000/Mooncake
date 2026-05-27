# Mooncake Store 日志 Trace 与异步输出改造说明

## 修改范围

本次改造围绕 Mooncake Store 的 get/put 读写链路：

- 新增 `mooncake-common/include/mooncake_logging.h` 和 `mooncake-common/src/mooncake_logging.cpp`。
- `real_client.cpp`、`client_service.cpp`、`transfer_task.cpp` 的关键日志从 `LOG/VLOG` 切换为 `MC_LOG/MC_VLOG`。
- `mooncake_common` 新增日志实现源文件，并让 `mooncake_store` 链接 `mooncake_common`。
- `MC_LOG_ENABLE=off` 会同步影响 transfer engine 的 `FLAGS_minloglevel`，用于关闭未迁移到 `MC_LOG` 的 glog 普通日志。

## TraceId

每个 RealClient 入口自动创建一个进程内唯一的 `trace_id`，不改变公开 API：

- get 路径：`get_buffer`、`batch_get_buffer`、`get_into`、`batch_get_into`。
- put 路径：`put`、`put_batch`、`put_parts`、`put_from`、`batch_put_from`、`batch_put_from_multi_buffers`。

`trace_id` 由进程 ID、启动后的 steady clock 时间片和原子递增计数组合生成。同步调用链通过 thread-local 传递；异步路径在提交任务时捕获当前 trace，并在 worker 线程中用 `ScopedTraceId` 恢复。

日志格式统一带：

```text
trace_id[123456789] get_into_breakdown key[k1] query_us[10] select_us[2] read_us[80] total_us[95] type[MEMORY] mode[full] status[read_ok]
```

如果日志不在请求上下文中，trace 字段为：

```text
trace_id[none] ...
```

## 日志开关

新增环境变量：

```bash
MC_LOG_ENABLE=on   # 默认，输出日志
MC_LOG_ENABLE=off  # 关闭 INFO/WARNING/ERROR 普通日志
```

可接受的关闭值包括 `off`、`0`、`false`、`no`。`FATAL` 不受关闭影响。

## 异步 glog 输出

`MC_LOG` 默认异步输出，不额外增加 `MC_LOG_ASYNC` 开关：

1. 业务线程构造日志消息。
2. `AsyncLogMessage` 析构时把 `severity/file/line/trace_id/message` 放入有界队列。
3. 单后台线程消费队列，并调用 glog 落盘或输出。
4. 队列容量固定为 8192；队列满时阻塞，优先保证日志不丢。
5. 进程退出时通过 `atexit` flush 队列。

## TransferData Wait 首次耗时定位

没有在 `store_py.cpp::PYBIND11_MODULE(store, m)` 加 `MC_IMPORT_TRACE` 或 import 阶段计时。

为了定位第一次 `TransferData::Wait` 很长的问题，本次在 transfer 首次执行路径增加拆解日志：

- `client_create_breakdown`：拆 `ConnectToMaster`、`GetStorageConfig`、`InitTransferEngine`、`InitTransferSubmitter`。
- `open_segment_breakdown`：记录 `openSegment` endpoint、耗时和状态。
- `submit_transfer_breakdown`：记录 `allocateBatchID`、`submitTransfer`、batch id、request 数和是否首次 transfer。
- `transfer_future_wait`：记录 `future->get()` 等待耗时、策略、是否首次 wait。
- `transfer_data`：记录 submit/wait 总拆分、策略、读写方向和结果。

使用方法：连续执行两次相同 get/put，对比第一次和第二次的上述日志。如果第一次主要长在 `open_segment_breakdown` 或 `submit_transfer_breakdown`，说明更接近 transfer/连接 lazy init；如果长在 `transfer_future_wait`，继续看底层 transport 完成路径。
