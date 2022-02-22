title:: Rust/std/Error

- `Error` 是 [[Rust]] 标准库提供的 Error trait
	- ```rust
	  pub trait Error: Debug + Display {
	      fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
	      fn backtrace(&self) -> Option<&Backtrace> { ... }
	      fn description(&self) -> &str { ... }
	      fn cause(&self) -> Option<&dyn Error> { ... }
	  }
	  ```
	- 所有的方法都提供了内置的实现，所以任意实现了 `Debug + Display` 的类型都能够作为 Error 返回。
	- 这里的 `description` 和 `cause` 都已经被弃用了，所以不需要考虑
		- `description` 被 `Display` 替代
		- `cause` 被 `source` 替代
- `source` 旨在返回当前的报错原因
	- 比如我们可以在自己包装的 error 结构体中包装一个 `io::Error`，并在 `source` 中返回
- `backtrace` 旨在返回一个 stack backtrace，用来指示错误发生在哪里
	- 目前这个 API 还是 unstable 的，需要 feature flag `backtrace` 来开启
-
- 大多数时候我们不需要自己来实现 Error trait， [[dtolnay]] 开发了 [[dtolnay/thiserror]] 和 [[dtolnay/anyhow]] 分别处理两种常见的情况：
	- [[dtolnay/anyhow]] 用在很多应用中，我们不关心底层返回的 error 类型，只需要直接返回给调用者
		- 使用 `anyhow::Result` 作为 Result 类型
			- ```rust
			  use anyhow::Result;
			  
			  fn get_cluster_info() -> Result<ClusterMap> {
			      let config = std::fs::read_to_string("cluster.json")?;
			      let map: ClusterMap = serde_json::from_str(&config)?;
			      Ok(map)
			  }
			  ```
		- 使用 `anyhow!` 来快速返回一个 Error
			- ```rust
			  use anyhow::{anyhow, Result};
			  
			  fn lookup(key: &str) -> Result<V> {
			      if key.len() != 16 {
			          return Err(anyhow!("key length must be 16 characters, got {:?}", key));
			      }
			  
			      // ...
			  }
			  ```
	- [[dtolnay/thiserror]] 用在 lib 中，通过过程宏的方式生成符合要求的 Error：
		- ```rust
		  use thiserror::Error;
		  
		  #[derive(Error, Debug)]
		  pub enum DataStoreError {
		      #[error("data store disconnected")]
		      Disconnect(#[from] io::Error),
		      #[error("the data for key `{0}` is not available")]
		      Redaction(String),
		      #[error("invalid header (expected {expected:?}, found {found:?})")]
		      InvalidHeader {
		          expected: String,
		          found: String,
		      },
		      #[error("unknown data store error")]
		      Unknown,
		  }
		  ```
		- 支持 `#[from]` 来自动实现 `impl From<io::Error> for YourError`.
		- 支持