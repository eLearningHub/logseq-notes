title:: Rust/std/macro_rules

- 使用 `macro_rules` 来为每个服务自动生成测试
	- ```rust
	  macro_rules! behavior_tests {
	      ($($service:ident),*) => {
	          $(
	              behavior_test!($service, test_normal, test_stat_root, test_stat_non_exist);
	          )*
	      };
	  }
	  
	  macro_rules! behavior_test {
	      ($service:ident, $($test:ident),*) => {
	          paste::item! {
	              mod [<$service>] {
	                  use super::*;
	  
	                  $(
	                      #[tokio::test]
	                      async fn [< $test >]() -> Result<()> {
	                          init_logger();
	  
	                          let acc = super::services::$service::new().await?;
	                          if acc.is_none() {
	                              return Ok(())
	                          }
	                          super::$test(Operator::new(acc.unwrap())).await
	                      }
	                  )*
	              }
	          }
	      };
	  }
	  
	  behavior_tests!(azblob, fs, memory, s3);
	  ```
- 支持捕获 `#[cfg(abc= xxx)]` 这样的 meta
	- ```rust
	  macro_rules! behavior_tests {
	      ($($service:ident),*) => {
	          $(
	              behavior_test!(
	                  $service,
	  
	                  #[cfg(feature = "compress")]
	                  test_read_decompress_gzip,
	                  #[cfg(feature = "compress")]
	                  test_read_decompress_gzip_with,
	              );
	          )*
	      };
	  }
	  
	  macro_rules! behavior_test {
	      ($service:ident, $($(#[$meta:meta])* $test:ident),*,) => {
	          paste::item! {
	              mod [<$service>] {
	                  use super::*;
	  
	                  $(
	                      #[tokio::test]
	                      $(
	                          #[$meta]
	                      )*
	                      async fn [< $test >]() -> Result<()> {
	                          init_logger();
	  
	                          let acc = super::services::$service::new().await?;
	                          if acc.is_none() {
	                              return Ok(())
	                          }
	                          super::$test(Operator::new(acc.unwrap())).await
	                      }
	                  )*
	              }
	          }
	      };
	  }
	  ```
-
- 注意这里的限制比较严格，一定要先声明才能使用
	- 最好把生成的代码放在一个 mod 里面，避免受到外界 use 和重名函数的影响
	- 比较好的实践是尽可能使用绝对引用路径
-
- 使用 cargo expand 来调试
	- ```shell
	   cargo expand --test behavior
	  ```
-
- 参考资料
	- [Macros in Rust: A tutorial with examples](https://blog.logrocket.com/macros-in-rust-a-tutorial-with-examples/)
	- [Procedural Macros](https://doc.rust-lang.org/reference/procedural-macros.html)
-