type:: [[Project]]
source:: [dtolnay/async-trait](https://github.com/dtolnay/async-trait)

- 原理
	- 将下列的 async 函数转换为 `Pin<Box<dyn Future + Send + 'async_trait>>`
	- ```rust
	  #[async_trait]
	  trait Advertisement {
	      async fn run(&self);
	  }
	  
	  struct AutoplayingVideo {
	      media_url: String,
	  }
	  
	  #[async_trait]
	  impl Advertisement for AutoplayingVideo {
	      async fn run(&self) {
	          let stream = connect(&self.media_url).await;
	          stream.play().await;
	  
	          // Video probably persuaded user to join our mailing list!
	          Modal.run().await;
	      }
	  }
	  ```
	- ```rust
	  impl Advertisement for AutoplayingVideo {
	      fn run<'async_trait>(
	          &'async_trait self,
	      ) -> Pin<Box<dyn std::future::Future<Output = ()> + Send + 'async_trait>>
	      where
	          Self: Sync + 'async_trait,
	      {
	          async fn run(_self: &AutoplayingVideo) {
	              /* the original method body */
	          }
	  
	          Box::pin(run(self))
	      }
	  }
	  ```