- [[serfig]] 肯定需要支持从 [[toml]]/[[yaml]] 中加载配置，需要考虑一下 API 如何设计
-
- config-rs 的设计是暴露了一个统一的 File 结构体
	- `pub struct File<T, F> { /* private fields */ }`
	- 好处在于可以很方便的支持给定一个 base name，然后自动探测支持的 file format 这种操作
	- ```rust
	  pub fn with_name(name: &str) -> Self
	  ```
-
- 考虑一下 serfig 支持什么样的功能？
	- 考虑核心定位：Layered configuration system built upon serde
		- 需要能够跟 serde 进行完美的集成
	- 在这个基础上，显然需要支持的是
		- 支持各种各样基于 serde 的格式：env/toml/yaml/json 等等
		- 提供一个方便的 api 使得用户可以轻松的添加多种格式的支持
	- 在这个基础上，有没有什么可以延拓的边界？
		- live watch / reloading？
			- 看起来不错，但是前期可以不考虑
		- async ？
			- 需要支持，不过前期不考虑
		- load from http？
			- 依赖 async，同样之后再说
			- 同时也要看用户的需求，我们会需要使用 http 来加载配置吗？
		- 通过特定的语法来获取某项配置
			- 比如 `cfg.get("a.b.c")`
			- 不考虑，用户应该始终使用 serde 来解析配置
-
- 现在看起来是这样
	- ```rust
	  let cfg = Builder::default().with_collector(Environment);
	  let t: TestConfig = cfg.build().expect("must success");
	  ```
	- `with_collector` => `collect`
-
- 如果 serfig 没有自己的 file 类型，感觉非常不方便
	- 每种格式都需要自行处理文件相关的操作
- 实际上格式和 file 应该是正交关系
	- env 是因为格式和 source 是同一个，所以放在一起就行
- 每个 collector 都应该拆分为 source & format
	- source 和 format 是正交关系
- 可能的 source 包括
	- file / io::reader / str
		- 返回一个 bytes，是否合法由 format 判定
- 可能的 format 包括
	- json/toml 等等
- 这样就可以自动给 source + format 实现 collector 了
-