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
						- 或者不提供
-
- 用户体验
	- op.objects("xxxxx") 得到一个 ObjectStream，然后就可以不断的 next 来获取 object 直到全部返回
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