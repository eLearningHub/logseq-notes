title:: Rust/Cargo/Features

- features 用于支持条件编译和可选依赖，用户可以选择性开启部分特性以避免编译不需要使用的依赖。
-
- 增加 feature
	- ```toml
	  [features]
	  # Defines a feature named `webp` that does not enable any other features.
	  webp = []
	  ```
	- 在代码中使用条件编译
		- ```rust
		  // This conditionally includes a module which implements WEBP support.
		  #[cfg(feature = "webp")]
		  pub mod webp;
		  ```
	- 在依赖中启用可选依赖
		- ```toml
		  [dependencies]
		  gif = { version = "0.6.3", optional = true }
		  ```
		- 在启用可选依赖之后，这个依赖就会变成一个同名的 feature
		- 可以通过条件编译的形式来控制：
			- `cfg(feature = "gif")`
		- 或者作为其他特性的依赖：
			- ```toml
			  [dependencies]
			  ravif = { version = "0.6.3", optional = true }
			  rgb = { version = "0.8.25", optional = true }
			  
			  [features]
			  avif = ["ravif", "rgb"]
			  ```
- 默认 features
	- 所有的 features 中 `default` 是特殊的
	- ```toml
	  [features]
	  default = ["ico", "webp"]
	  bmp = []
	  png = []
	  ico = ["bmp", "png"]
	  webp = []
	  ```
	- 依赖这个 crate 的用户会默认启用所有的 `default` 特性
	- 可以通过 `--no-default-features` 命令行参数或者配置依赖 `default-features = false` 来关闭
		- ```toml
		  [dependencies]
		  flate2 = { version = "1.0.3", default-features = false, features = ["zlib"] }
		  ```
- 为 features 增加文档
	- 最好在文档中把支持的 feature 及其各自的含义都在文档中列出来，否则用户很难发现
	- 参考 regex
		- https://github.com/rust-lang/regex/blob/1.4.2/src/lib.rs#L488-L583
		- https://docs.rs/regex/latest/regex/index.html#crate-features
-
- 一些注意事项
	- 每个 crate 定义的 feature 是互相独立的
		- > Enabling a feature on a package does not enable a feature of the same name on other packages.
	- 修改 default features 是 breaking change
		- 增减 default features 可能会 break 下游依赖的构建，需要谨慎处理
-
- 参考资料
	- [The Cargo Book: Features](https://doc.rust-lang.org/cargo/reference/features.html)