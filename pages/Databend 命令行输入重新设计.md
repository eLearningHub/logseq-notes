- clap 只支持从 args / env 输入，不支持从文件读取
	- 需要做不少额外的工作
-
- 需要一个库能够支持从多种 source 读取 args，并且可以跟 clap 完美集成
-
- 现在是在 databend 的入口处读取的数据
-
- 现在 config 有
	- load_from_args
	- load_from_file
	- load_from_env
-
- 首先要把 App 跟 config 分开
-
- 然后呢，config 怎么处理？有没有什么更加好的方案？
-
- 感觉不需要自己造轮子，config 已经非常好用了
-
- 那么问题来了，config 怎么跟 clap 集成好呢？是不是需要重复的两遍？
	- 好像不是很难啊
	- 把 Config 作为 App 的 field，实现 clap::Arg
	- 然后再去 config load
		- 优先从 clap load，然后 config 来load
		- 好像不太行？
- 好像需要给 config 加一个功能，允许任意 struct 作为它自己的 source
	- 是不是需要过程宏了？
	- 好像也不用，是不是可以给 Value 实现一个 serialize 就行？
		- 允许任意合法的结构体变成 Value
			- 哦，不行，Value 本身不是 Source
		- Config 是一个 Source
- 搞一把试试
-
- 有点蛋疼，给 Value 实现 serializer 不是非常爽
	- 感觉需要一个比 serde 更舒服一些的，不是在格式之间转换，而是在结构体之间转换的东西
-
- 比如说，我有一个 map，我要转换成一个结构体
- 现在做法是只能 From/Into，或者转换成某种中间的形式
	- 标准库里面也有一大堆 IntoXxxx 这种东西
- 有没有什么东西能够只需要实现特定 trait 就能做转换？
	- 这个东西是不是就叫 From？。。
		- 感觉还是不太一样的，想一想差别在哪里
- 如何支持嵌套的类型？
	- `HashMap<String, T>`
- 需要一个 serde value？
	- Config serialize -> Value
	- Value
- 应该是任意结构体都可以自动实现实现 From<x> for Value / From<value>  for X
	- 这样的话，只需要给结构体加上，就可以无缝转换了？
	- Config -> value -> config::Value
		- 跟 serde 的工作会重复吗？
			- 区别是不是纯内存转换？
			- 不处理纯文本格式？只支持结构体？
		- 需要做一整套基础设施吗？还是只需要实现 serde 的全部接口就行？
			- 是不是重复了 https://github.com/arcnmx/serde-value 的工作？
			- 有什么是 serde-value 没有做好的吗？
		- 要做一个 serde-bridge
			- 目标是实现 `From<impl Serialize> For impl Deserialize` ？
			- Config -> value
				- let v = config.serialize()
			- value -> config::Value
				- let cv = value.deserialize();
	- 先不去想这些，考虑一下，如果我要实现 A -> B，该怎么做
		- A serialize 成 Value
			- `From<impl Serialize> for Value`
		- 然后 B 可以从 value 中 deserialize
			- `From<Value> for impl Deserialize`
-
- Value 本身需要是可序列化的吗？好像不是
	- Value 就是一个表达格式，本身不具有含义
-
- struct -> map
-
- serde-raw？
-
-
- 所以问题出在 config-rs 实现是错的，没必要给 Value 实现 Deserialize
	- 有了 serde-bridge 之后是不是就不需要 config-rs 自己的那套抽象了？
	- Clap -> Config -> value -> map
-
- ---
- [[2022-04-20]]
-
- serfig 已经基本上能用了，下面考虑一下如何用到 databend 里面
- ```rust
  pub struct Config {
      #[clap(long, short = 'c', env = CONFIG_FILE, default_value = "")]
      pub config_file: String,
  
      // Query engine config.
      #[clap(flatten)]
      pub query: QueryConfig,
  
      #[clap(flatten)]
      pub log: LogConfig,
  
      // Meta Service config.
      #[clap(flatten)]
      pub meta: MetaConfig,
  
      // Storage backend config.
      #[clap(flatten)]
      pub storage: StorageConfig,
  }
  ```
- ```rust
  pub struct StorageConfig {
      /// Current storage type: fs|s3
      #[clap(long, env = STORAGE_TYPE, default_value = "fs")]
      pub storage_type: String,
  
      #[clap(long, env = STORAGE_NUM_CPUS, default_value = "0")]
      pub storage_num_cpus: u64,
  
      // Fs storage backend config.
      #[clap(flatten)]
      pub fs: FsStorageConfig,
  
      // S3 storage backend config.
      #[clap(flatten)]
      pub s3: S3StorageConfig,
  
      // azure storage blob config.
      #[clap(flatten)]
      pub azure_storage_blob: AzureStorageBlobConfig,
  }
  ```
- 考虑一下如何同时支持 clap & load_from_file
	- 先解析 env，然后解析 file，最后 load via clap
-