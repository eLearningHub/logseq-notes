- 现在 backon 只实现了 backoff 的算法，但是没有直接支持 retry
- 用户需要这样使用
	- ```rust
	  use backon::ExponentialBackoff;
	  use anyhow::Result;
	  
	  #[tokio::main]
	  async fn main() -> Result<()> {
	      for delay in ExponentialBackoff::default() {
	          let x = reqwest::get("https://www.rust-lang.org").await?.text().await;
	          match x {
	              Ok(v) => {
	                  println!("Successfully fetched");
	                  break;
	              },
	              Err(_) => {
	                  tokio::time::sleep(delay).await;
	                  continue
	              }
	          };
	      }
	  
	      Ok(())
	  }
	  ```
- 太呆了，需要想一下比较合理的方式是怎么样
	- 返回一个 ControlFlow？
		- 先不要考虑具体的实现，先想想用户会需要如何使用
-
- retry 特定的请求
	- ```rust
	  let backoff = ExponentialBackoff::default();
	  let s = reqwest::get("https://www.rust-lang.org").retry(backoff).await?.text().await;
	  ```
- retry 一整个 async block？
	- 这个可以晚点再说
-
- 先看一下如何 retry 特定 feature
	- 好像不行啊- -，Rust 类型系统不让这么玩
	- 问题在于想要 retry 的话，对这个 future 返回的 Output 是有要求的
		- 需要是 Result，然后 Err 还得是 Retryable error
	- 啊，有 TryFuture
		- ```rust
		  #[pin_project]
		  struct Retry<T: TryFuture<Error = Err>, B: Policy, Err: RetryableError> {
		      #[pin]
		      inner: T,
		      backoff: B,
		  }
		  
		  ```
-
- 擦，是我傻逼了
	- future return Ready(Err(_)) 之后显然不能再去 poll 了，除非我 clone 这个 future
	- 这也是为什么现有的实现全都要求传入一个闭包的原因，因为我需要重新构建这个 future