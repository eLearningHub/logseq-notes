- [[OpenDAL]] 肯定需要支持 List
-
- fs 的 List 和对象存储的 List 是不一样的
	- fs 需要操作同一个 fd，不停的 getdents 然后解析
		- 最好对 fs 能保持使用同一个 fd，注意这个 read_dir 操作是不能再入的，用户不能中断之后再恢复
	- 对象的每次 list 都是一个新的请求，返回一组(一般为 100 个) objects
		- 对象会返回一个 continuation_token，下次请求的时候再带上
			- 根据实现不同，这个 continuation_token 也不一定是持久化的，用户需要尽量避免保存它
		- 其他考虑
			- 对象的 list 会返回更多信息，比如 content_length 等等，有没有可能 cache 这些数据避免重复访问
			- 把 Metadata 放进 object？Metadata 不对外暴露接口？
				- 遇到的问题
					- `Object::metadata()` 要求签名变成 `&mut self` 了，有点奇怪
						- 把所有函数都改成要求 `&mut self` 如何？
					- 用 `RefCell`？
						- 感觉非常的僵硬，语义也不太对，还有可能出现 runtime panic
							- ```rust
							  pub async fn metadata(&self) -> Result<Metadata> {
							    let mut meta = self.meta.borrow_mut();
							    if meta.complete {
							      return Ok(meta.clone());
							    }
							  
							    let op = &OpStat::new(&self.path());
							    *meta = self.acc.stat(op).await?;
							  
							    Ok(meta.clone())
							  }
							  ```
-
- 用户体验
	- op.objects("xxxxx") 得到一个 ObjectStream，然后就可以不断的 next 来获取 object 直到全部返回
	- object.metadata() 可以返回所有的 metadata
		- 支持 reuse list 时返回的 object 信息，比如 content_length 等等
		- 在获取指定的 meta 信息时，可以自动的去发出 stat 请求
-
- 问题
	- 只允许 List 一个 dir 吗？
		- 如果有这个规则限制的话，对象这边可以做一些强制的检查，比如 list 之前先 stat 一下
			- 或者对 path name 做一些处理：`xxxx` -> `xxxx/`
	- 是否需要支持 prefix list？
		- 没必要支持
	- ObjectStream 每次 `poll_next` 是返回一个 object 还是一组？
		- 暴露给用户的是一个？底层使用的是一组？
- 设计
	- object 中需要携带 mode 来标记这是 dir 还是 file，抑或是 link
	- 需要保持 fd 的话，底层的就需要返回一个类似 Reader 的东西
		- 让底层返回一个 `BoxedObjectStream = Box<dyn futures::Stream<Item = Result<Object>>`
	- ```rust
	  pub type BoxedObjectStream = Box<dyn futures::Stream<Item = Result<Object>>>;
	  
	  pub struct ObjectStream {
	      acc: Arc<dyn Accessor>,
	      path: String,
	  
	      state: State,
	  }
	  
	  enum State {
	      Idle,
	      Sending(BoxFuture<'static, Result<BoxedObjectStream>>),
	      Listing(BoxedObjectStream),
	  }
	  
	  impl futures::Stream for ObjectStream {
	      type Item = Result<Object>;
	  
	      fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
	          todo!()
	      }
	  }
	  
	  ```
- 尝试
	- 将 Metadata 直接嵌入 Object
		- ```rust
		      async fn fetch(&mut self) -> Result<()> {
		          if self.complete {
		              return Ok(());
		          }
		  
		          let m = self.acc.stat(&OpStat::new(&self.path)).await?;
		          self.merge(&m);
		  
		          Ok(())
		      }
		  
		      pub fn path(&self) -> &str {
		          &self.path
		      }
		  
		      pub async fn mode(&mut self) -> Result<ObjectMode> {
		          if let Some(v) = self.mode {
		              return Ok(v);
		          }
		  
		          self.fetch().await?;
		          if let Some(v) = self.mode {
		              return Ok(v);
		          }
		  
		          unreachable!("object meta should have mode, but it's not")
		      }
		  ```
		- 弊端
			- 所有的 getter 方法都要 `&mut self`，都需要 async，都需要处理错误
			- 没法区分朴素的 getter 和支持 fetch 的 getter
				- 增加一个 direct_get_xx ？
			-