NOTE: I am mostly writing this down to present to someone who is already familiar with the system, but I have laid down some ground work to make it slightly better. Write up is also heavily code referential, so sorry if that's not up your alley.

[RisingWave](https://github.com/risingwavelabs/risingwave) is a popular and open-source streaming database, it can work with a variety of different sources and sinks and has capabilities to provide performant real time analyses on streaming data along side service ad-hoc queries. Basically a lot of buzzwords.

I have grown interest into the system and was trying to understand how it prevents data loss with so many different sinks in case one of it's compute node dies? We will be looking into handling of iceberg sink cause that's what I am working with these days. I am going to assume familiarity with iceberg already cause understanding that would take another several blog posts.

One of the good features of iceberg is it's decoupling between data files and metadata files. One can take existing parquet files and create a table out of them easily. Work for the [same](https://github.com/apache/iceberg-rust/issues/932) is active in iceberg-rust. Even when comitting iceberg writers do the same, write data files first and then try to write metadata files, if they fail (they may fail cause another writer's commit would cause ACID guarantees to fail on table) they just have to re-generate metadata files and try to commit again.

So writing data files vs committing are separate processes, same happens in iceberg-rs and hence RisingWave, for iceberg sink these are the locations where each occurs:  
- Writing happens under `IcebergSinkWriter` [here](https://github.com/risingwavelabs/risingwave/blob/1a6eb0001c806c547de129d4cf66035ec66e4fe1/src/connector/src/sink/iceberg/mod.rs#L863)
- Commiting happens [here](https://github.com/risingwavelabs/risingwave/blob/1a6eb0001c806c547de129d4cf66035ec66e4fe1/src/connector/src/sink/iceberg/mod.rs#L1208)

Getting back to RisingWave, core idea of persisting such state in databases is to use some kind of logs, a lot of databases have their own WAL implementation. RisingWave also leverages concept of log stores for the same. [These](https://github.com/risingwavelabs/risingwave/tree/main/src/stream/src/common/log_store_impl) are the current log stores implementation:
- In Memory Log Store
- KV Log Store

Now our doubt was what if RisingWave compute node crashes before commit happens. LogStores implements `LogReader`. `LogReader` abstraction shows what [all methods](https://github.com/risingwavelabs/risingwave/blob/1a6eb0001c806c547de129d4cf66035ec66e4fe1/src/connector/src/sink/log_store.rs#L158C22-L179) does it provide, namely:
- `init`
- `next_item` , read next item in log
- `truncate` , increments read offset in log
- `rewind` , decrements read offset in log

These methods are used along side RisingWave's internal global clock to make sure no data is lost. Hierarchy of internal clock looks like this:  

- `barriers` every configurable ms, configurable using `barrier_interval_ms` in system params
- `checkpoints` every N barriers, configurable using `checkpoint_frequency` system param
- `commits` every N checkpoints, configurable for iceberg sink using `commit_checkpoint_interval`

So we can keep reading data async on every barrier using `next_item` and keep `truncate`ing on every commit. This would ensure we lose no data for different types of sinks.

Let's see what happens for iceberg sink:

Firstly, how `LogReader` relates to our iceberg writer. `LogReader` is used by `LogSinker` , in our case `DecoupleCheckpointLogSinker` [here](https://github.com/risingwavelabs/risingwave/blob/5dcc141cf86b5b41e7e6965ac7ec840c73aad247/src/connector/src/sink/decouple_checkpoint_log_sink.rs#L80) and that finally calls:  

- For writing:  `write_batch` [here](https://github.com/risingwavelabs/risingwave/blob/5dcc141cf86b5b41e7e6965ac7ec840c73aad247/src/connector/src/sink/decouple_checkpoint_log_sink.rs#L138)
- For committing: `commit` [here](https://github.com/risingwavelabs/risingwave/blob/1a6eb0001c806c547de129d4cf66035ec66e4fe1/src/meta/src/manager/sink_coordination/coordinator_worker.rs#L272), this follows central clock of barriers

Now, `DecoupleCheckpointLogSinker` also listens to central clock of barrier and writes data files to object store on each barrier [here](https://github.com/risingwavelabs/risingwave/blob/1a6eb0001c806c547de129d4cf66035ec66e4fe1/src/connector/src/sink/iceberg/mod.rs#L980-L1090) (i.e. call `close` method on `data file writer`), but it actually commits the result on every N checkpoints.

So technically, if barrier and checkpoint values are not same and a compute node crashes between two checkpoints, we would have written data files to object store, but it would not be committed i.e. no metadata files. So these would fall under table maintenance job of `orphan files`.

This can be mitigated by simply setting `checkpoint_frequency`  to 1 i.e. trigger at every barrier and also `commit_checkpoint_interval` to 1 i.e. `commit` on every barrier/checkpoint.

Now, how to increase batching size? That can be done by configuring `barrier_interval_ms` . Though this could be a bad idea cause barriers are used internally for a lot of other things, they are like `ticks` in minecraft engine. So making everything slower for batching can make us lose other system internal state/data leaving system in weird non-recoverable condition.