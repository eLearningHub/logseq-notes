title:: OpenDAL 使用 io::Result

- 考虑到跟 Reader / Writer 更友好的交互，考虑所有函数都返回 io::Result
-
- 基本情况
	- ```rust
	  use std::io::{Error, ErrorKind};
	  
	  // errors can be created from strings
	  let custom_error = Error::new(ErrorKind::Other, "oh no!");
	  
	  // errors can also be created from other errors
	  let custom_error2 = Error::new(ErrorKind::Interrupted, custom_error);
	  
	  // creating an error without payload
	  let eof_error = Error::from(ErrorKind::UnexpectedEof);
	  ```
-
- 我们还是需要自行定义一个 Error 用来传递上下文信息
	- 区别在于我们不再需要自己的 kind
	- 这样的话还需要 thiserror 吗？
-
- 现在有三大类问题
	- Backend 相关 =>  io::ErrorKind::Other + BackendError?
		- 感觉没有必要额外抽象 error？
	- Object 相关 => 对应的 ErrorKind + ObjectError (op，path，cause)
	- Unexpected => Other + anyhow