# 导入总览

数据导入是指将原始数据按照业务需求进行清洗、转换、并加载到 StarRocks 中的过程，从而可以在 StarRocks  系统中进行极速统一的数据分析。

## 导入模式

StarRocks 支持两种导入模式：同步导入和异步导入。

> 说明：如果是外部程序接入 StarRocks 的导入，需要先判断使用哪种导入模式，然后再确定接入逻辑。

### 同步导入

同步导入是指用户创建导入作业，StarRocks 同步执行，执行完成后返回导入结果。您可以通过返回的导入结果判断导入作业是否成功。

支持同步模式的导入方式有 Stream Load 和 INSERT INTO。

导入过程如下：

1. 创建导入作业。

2. 查看 StarRocks 返回的导入结果。

3. 判断导入结果。如果导入结果为失败，可以重试导入作业。

### 异步导入

异步导入是指创建导入作业以后，StarRocks 直接返回创建成功，但创建成功不代表数据已经导入成功。StarRocks 会异步执行导入作业。在导入作业创建成功以后，您需要通过轮询的方式查看导入作业的状态。如果导入作业创建失败，可以根据失败信息，判断是否需要重试。

支持异步模式的导入方式有 Broker Load、Routine Load 和 Spark Load。

导入过程如下：

1. 创建导入作业。

2. 根据 StarRocks 返回的作业创建结果，判断作业是否创建成功。

   - 如果作业创建成功，进入步骤 3。
   - 如果作业创建失败，可以回到步骤 1，尝试重试导入作业。

3. 轮询查看导入作业的状态，直到状态变为 **FINISHED** 或 **CANCELLED**。

在异步的导入方式 Broker Load 和 Spark Load 中，一个导入作业的执行流程主要分为 5 个阶段，如下图所示。

![异步导入流程图](/assets/4.1.1.png)

每个阶段的描述如下：

1. **PENDING**<br>
   该阶段是指提交导入作业后，等待 FE 调度执行。

2. **ETL**<br>
   该阶段执行数据的预处理，包括清洗、分区、排序、聚合等。

   > 说明：如果是 Broker Load 作业，该阶段会直接完成。

3. **LOADING**<br>
   该阶段先对数据进行清洗和转换，然后将数据发送给 BE 处理。当数据全部导入后，进入等待生效过程，此时，导入作业的状态依旧是 **LOADING**。

4. **FINISHED**<br>
   在导入作业涉及的所有数据均生效后，作业的状态变成 **FINISHED**，此时，导入的数据均可查询。**FINISHED** 是导入作业的最终状态。

5. **CANCELLED**<br>
   在导入作业的状态变为 **FINISHED** 之前，您可以随时取消作业。另外，如果导入出现错误，StarRocks 系统也会自动取消导入作业。作业取消后，进入 **CANCELLED** 状态。**CANCELLED** 也是导入作业的一种最终状态。

## 导入方式

StarRocks 提供 [Stream Load](/loading/StreamLoad.md)、[Broker Load](/loading/BrokerLoad.md)、 [Routine Load](/loading/RoutineLoad.md)、[Spark Load](/loading/SparkLoad.md) 和 [INSERT INTO](/loading/InsertInto.md) 多种导入方式，满足您在不同业务场景下的数据导入需求。

| 导入方式           | 业务场景                                                     | 数据量（单作业）                 | 数据源                                       | 数据格式              | 同步模式 |
| ------------------ | ------------------------------------------------------------ | -------------------------------- | -------------------------------------------- | --------------------- | -------- |
| Stream Load        | 通过 HTTP 协议导入本地文件、或通过程序导入数据流。           | 10 GB 以内（CSV 格式不受此限制） | - 本地文件<br> - 程序                            | - CSV<br> - JSON          | 同步     |
| Broker Load        | 通过独立的 Broker 程序从外部云存储系统导入。                 | 数十到数百 GB                    | - HDFS<br> - Amazon S3<br> - 阿里云 OSS<br> - 腾讯云 COS<br> | - CSV<br> - ORC<br> - Parquet | 异步     |
| Routine Load       | 从 Apache Kafka® 等流式数据源实时地导入数据。                | 微批导入 MB 到 GB 级             | Kafka                                        | - CSV<br> - JSON          | 异步     |
| Spark Load         | - 通过 Apache Spark™ 集群初次从云存储系统迁移导入大量数据。<br> - 需要做全局数据字典来精确去重。 | 数十 GB 到 TB级别                | - HDFS<br> - Hive                                | - CSV<br> - Apache Hive™  | 异步     |
| INSERT INTO SELECT | - 外表导入。<br> - StarRocks 数据表之间的数据导入。              | 跟内存相关                       | - StarRocks 表<br> - 外部表                      | StarRocks 数据表      | 同步     |
| INSERT INTO VALUES | - 单条批量小数据量插入。<br> - 通过 JDBC 等接口导入。            | 简单测试用                       | - 程序<br> - ETL 工具                            | SQL                   | 同步     |

您可以根据业务场景、数据量、数据源、数据格式和导入频次等来选择合适的导入方式。另外，在选择导入方式时，可以注意以下几点：

- 从 Kafka 导入数据的时候，如果导入过程中有复杂的多表关联和 ETL 预处理，可以先使用 Apache Flink® 对数据进行处理，然后再通过 Stream Load 把数据导入到 StarRocks 中。StarRocks 提供标准的 [flink-connector-starrocks](/loading/Flink-connector-starrocks.md) 插件，可以帮助您把 Flink 中的数据导入到 StarRocks。

- 如果要导入 Hive 文件，除了可以使用 [Spark Load](/loading/SparkLoad.md) 和 [Broker Load](/loading/BrokerLoad.md) 以外，推荐使用 [Hive 外部表](/using_starrocks/External_table.md)的方式实现导入。

- 如果要导入 MySQL 数据，除了可以使用 [starrockswriter](/loading/DataX-starrocks-writer.md) 以外，推荐通过 [MySQL 外部表](/using_starrocks/External_table.md)的方式，使用 INSERT INTO SELECT 语句实现导入。

- 对于 Oracle、PostgreSQL等数据源，推荐使用 [starrockswriter](/loading/DataX-starrocks-writer.md) 实现导入。如果要导入的数据源可以通过 Broker 或 Spark 程序读取，也可以采用 [Spark Load](/loading/SparkLoad.md) 或 [Broker Load](/loading/BrokerLoad.md) 实现导入。

下图详细展示了在各种数据源场景下，应该选择哪一种导入方式。

![数据源与导入方式关系图](/assets/4.1.2.png)

## 数据类型

StarRocks 支持导入所有数据类型。个别数据类型的导入可能会存在一些限制，具体请参见[数据类型](/sql-reference/sql-statements/data-types/BIGINT.md)。

## 使用说明

向 StarRocks 导入数据时，除了依次选择合适的导入方式、导入协议和导入模式以外，还需要注意以下事项：

- 制定标签 (Label) 生成策略。
  标签生成策略需满足对每一批次数据唯一且固定的原则。

- 保证“精确一次 (Exactly-Once) ”语义的实现。
  外部系统需要保证数据导入的“至少一次 (At-Least-Once) ”语义（一般是“失败时保持用相同的 Label 重试”即可），StarRocks 使用标签机制保证数据导入的“至多一次 (At-Most-Once) ”语义。这样整体上就可以实现数据导入的 “精确一次 ”语义。

## 内存限制

您可以通过设置参数来限制单个导入作业的内存使用，以防止导入作业占用过多内存而导致发生 OOM 异常，特别是在导入并发较高的情况下。同时，您也需要注意避免设置过小的内存使用上限，因为内存使用上限过小，导入过程中可能会因为内存使用量达到上限而频繁地将内存中的数据刷出到磁盘，进而可能影响导入效率。建议您根据具体的业务场景要求，合理地设置内存使用上限。

不同的导入方式限制内存的方式略有不同，具体可参考各导入方式的详细介绍章节。需要注意的是，一个导入作业通常都会分布在多个 BE 上执行，这些内存参数限制的是一个导入作业在单个 BE 上的内存使用，而不是在整个集群上的内存使用总和。

您还可以通过设置一些参数来限制在单个 BE 上运行的所有导入作业的总的内存使用上限。可参考本文“系统配置”章节。

## 系统配置

本节解释对所有导入方式均适用的参数配置。

### FE 配置

您可以通过修改每个 FE 的配置文件 **fe.conf** 来设置如下参数：

- `max_load_timeout_second` 和 `min_load_timeout_second`
  设置导入超时时间的最大、最小值，单位均为秒。默认的最大超时时间为 3 天，默认的最小超时时间为 1 秒。自定义的导入超时时间不能超过这个最大、最小值范围。该参数配置适用于所有模式的导入作业。

- `desired_max_waiting_jobs`
  等待队列可以容纳的导入作业的最大数目，默认值为 100。如果 FE 中处于 **PENDING** 状态的导入作业数目达到该值，FE 会拒绝新的导入请求。该参数配置仅对异步执行的导入有效。

- `max_running_txn_num_per_db`
  StarRocks 集群每个数据库中正在运行的导入作业的最大个数，默认值为 100。当数据库中正在运行的导入作业超过最大个数限制时，后续的导入不会执行。如果是同步的导入作业，作业会被拒绝；如果是异步的导入作业，作业会在队列中等待。

  - > 说明：所有模式的作业均包含在内、统一计数。

- `label_keep_max_second`
  已经完成、且处于 **FINISHED** 或 **CANCELLED** 状态的导入作业记录在 StarRocks 系统的保留时长，默认值为 3 天。该参数配置适用于所有模式的导入作业。

### BE 配置

您可以通过修改每个 BE 的配置文件 **be.conf** 来设置如下参数：

- `push_write_mbytes_per_sec`
  BE 上单个 Tablet 的最大写入速度，默认值为 10 MB/s。根据表结构 (Schema)、以及系统的不同，通常 BE 对单个 Tablet 的最大写入速度大约在 10 MB/s 到 30 MB/s 之间。可以适当调整该参数的取值来控制导入速度。

- `write_buffer_size`
  BE 上内存块的大小阈值，默认阈值为 100 MB。导入数据在 BE 上会先写入一个内存块，当内存块的大小达到这个阈值以后才会写回磁盘。如果阈值过小，可能会导致 BE 上存在大量的小文件，影响查询的性能，这时候可以适当提高这个阈值来减少文件数量。如果阈值过大，可能会导致远端程序呼叫（Remote Procedure Call，简称 RPC）超时，这时候需要配合下面的 `tablet_writer_rpc_timeout_sec` 参数来适当地调整 `write_buffer_size` 参数的取值。

- `tablet_writer_rpc_timeout_sec`
  导入过程中，Coordinator BE 发送一批次数据的 RPC 超时时间，默认为 600 秒。每批次数据包含 1024 行。RPC 可能涉及多个分片内存块的写盘操作，所以可能会因为写盘导致 RPC 超时，这时候可以适当地调整这个超时时间来减少超时错误（如 "send batch fail" 错误）。同时，如果调大 `write_buffer_size` 参数的取值，也需要适当地调大 `tablet_writer_rpc_timeout_sec` 参数的取值。

- `streaming_load_rpc_max_alive_time_sec`
  指定了 Writer 进程的等待超时时间，默认为 600 秒。在导入过程中，StarRocks 会为每个 Tablet 开启一个 Writer 进程，用于接收和写入数据。如果在参数指定时间内 Writer 进程没有收到任何数据，StarRocks 系统会自动销毁这个 Writer 进程。当系统处理速度较慢时，Writer 进程可能长时间接收不到下一批次数据，导致上报 "TabletWriter add batch with unknown id" 错误。这时候可适当调大这个参数的取值。

- `load_process_max_memory_limit_bytes` 和 `load_process_max_memory_limit_percent`
  用于导入的最大内存使用量和最大内存使用百分比，用来限制单个 BE 上所有导入作业的内存总和的使用上限。StarRocks 系统会在两个参数中取较小者，作为最终的使用上限。

  - `load_process_max_memory_limit_bytes`：指定 BE 上最大内存使用量，默认为 100 GB。
  - `load_process_max_memory_limit_percent`：指定 BE 上最大内存使用百分比，默认为 30%。该参数与 `mem_limit` 参数不同。`mem_limit` 参数指定的是 StarRocks 集群的总内存使用百分比，也就是物理内存使用百分比，默认值为 80%。
    假设 BE 所在机器物理内存大小为 M，则用于导入的内存上限为：M x 80% x 30%。

## 常见问题

请参见[导入常见问题](https://docs.starrocks.com/zh-cn/main/faq/loading/Loading_faq)。
