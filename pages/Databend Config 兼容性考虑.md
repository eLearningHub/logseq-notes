- [[Databend]] 已经合并了 RFC：[Config Backward Compatibility](https://github.com/datafuselabs/databend/pull/5324)
-
- 需要考虑的地方
	- config 格式兼容
	- env 变量兼容
	- args 兼容
	- 旧配置的提示，升级
	- protobuf 中的兼容
- 设计
	- Versioned Config
		- 解析的时候使用 version
		- 内部逻辑使用统一的 Config
- 需要先做个 demo 试试看
	- 如何保证 protobuf 兼容呢
		- 需要保留所有的字段？
		- 加上字段的前缀？
		- 在别的字段升级的时候一起清理掉旧的
	- 定义一个 Trait 叫做 VersionedStuct？
	- 是不是要每个结构体都自己实现一遍？
		- 应该是一个 version，一个 inner
			- enum？
	- 对外暴露一个统一的大结构体，然后内部包装成 enum？
- 决策点
	- 每个 public config 都需要加 version 吗？
	- config version 和 protobuf 是什么关系？
		- 分开处理更好些，因为不是所有的 config 都会成为 protobuf 的一部分
		-
- 变成一个 trait？
	- 使用起来会比较麻烦
- 主要是两个二进制
	- query & meta
- configs crate 负责 expose 当前的 config？
- 旧版本需要自行转换到最新的 config？
- 增加一个 Version 结构体，专门负责读取当前的配置版本
- 所有的 leaf config 都藏在 crate 里面，所有的 non-leaf config 都需要带有 version 结构体
	- leaf config 不直接导出？
		- 出现破坏性升级的时候，重构是不是比较难做
- 要不要把 QueryConfig 放进 config 库？
	- 感觉能行
- 要求每个提供 config 的库都实现 versioned
	- storage 拆到 common-io
	- log 拆到 common-tracing？
	- QueryConfig 放进 query
- 需要某种统一的机制来防止 config 出现 break
- 有个例外是
	- RpcClientTlsConfig
		- meta 和 query 都用到了他，但是参数不一样
			- rpc_tls_meta_server_root_ca_cert -> rpc_tls_server_root_ca_cert
			- rpc_tls_query_server_root_ca_cert -> rpc_tls_server_root_ca_cert
	- 这种好像没办法
- 维护 internal config 和 outer config
	- inner config 只需要关注自己的逻辑，内部所有的 config 相关依赖都从 inner config 读取
	- outer config 暴露给用户，需要处理 args 等一系列逻辑
	- 由 outer config 处理 outer 到 inner 的转换，inner 完全无感知
- outer config 主要是包括如下几处
	- config/env/args
	- protobuf
- 然后 outer config 会怎么实现呢
	- 一个独立的 version 字段
	- 先读取 version，然后根据 version 选择加载的结构体，最后再转换成 inner config
-
- ## 具体实现
-
- [[2022-05-16]]
- ```rust
  let mut storage_config = config.storage;
  storage_config.s3.access_key_id = mask_string(&storage_config.s3.access_key_id, 3);
  storage_config.s3.secret_access_key = mask_string(&storage_config.s3.secret_access_key, 3);
  storage_config.azblob.account_name = mask_string(&storage_config.azblob.account_name, 3);
  storage_config.azblob.account_key = mask_string(&storage_config.azblob.account_key, 3);
  let storage_config_value = serde_json::to_value(storage_config)?;
  ConfigsTable::extract_config(
    &mut names,
    &mut values,
    &mut groups,
    &mut descs,
    "storage".to_string(),
    storage_config_value,
  );
  ```
	- config tables 会依赖内部的 config 结构，按照目前的设计，config layout 可能会有变化
	- 可能需要 into outer？
-
-