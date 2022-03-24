title:: Rust/std/Drop

- [[Rust]] 通过 `Drop` trait 来实现内存空间的清理
-
- ```rust
  pub trait Drop {
      fn drop(&mut self);
  }
  ```
-
- drop 通常是由编译器来自动处理，手动调用类型提供的 drop 方法会有编译报错
	- ```rust
	  $ cargo run
	     Compiling drop-example v0.1.0 (file:///projects/drop-example)
	  error[E0040]: explicit use of destructor method
	    --> src/main.rs:16:7
	     |
	  16 |     c.drop();
	     |     --^^^^--
	     |     | |
	     |     | explicit destructor calls not allowed
	     |     help: consider using `drop` function: `drop(c)`
	  
	  For more information about this error, try `rustc --explain E0040`.
	  error: could not compile `drop-example` due to previous error
	  ```
	- 因为编译器会在这个值离开作用域的时候自动调用 drop，导致 double free 错误
- 取而代之的是我们可以使用 `std::mem::drop`
	- ```rust
	  pub fn drop<T>(_x: T) { }
	  ```
	- 通过把这个值直接 move 出去实现提前释放
-
- Rust 内部所有的智能指针和资源都通过 drop 实现了资源释放，比如 `Box<T>` 和 `OwnedFd`
	- ```rust
	  #[unstable(feature = "io_safety", issue = "87074")]
	  impl Drop for OwnedFd {
	      #[inline]
	      fn drop(&mut self) {
	          unsafe {
	              // Note that errors are ignored when closing a file descriptor. The
	              // reason for this is that if an error occurs we don't actually know if
	              // the file descriptor was closed or not, and if we retried (for
	              // something like EINTR), we might close another valid file descriptor
	              // opened after we closed ours.
	              let _ = libc::close(self.fd);
	          }
	      }
	  }
	  ```
	- 需要注意的是 drop 不会返回错误(显然），所以我们没有办法拿到文件在 close 时返回的错误
	- 所以如果我们要确保文件成功写入的话，需要手动执行 `flush` 或者 `sync_all` 并检查错误
		- > Files are automatically closed when they go out of scope. Errors detected on closing are ignored by the implementation of Drop. Use the method sync_all if these errors must be manually handled.
-
- 可以用来做测试资源的清理
	- 比如说我们可以在结构体中维护一些内部状态信息
	- ```rust
	  pub struct TempData {
	      op: Operator,
	      path: String,
	  }
	  
	  impl TempData {
	      pub fn generate(op: Operator, path: &str, content: Vec<u8>) -> Self {
	          TOKIO.block_on(async {
	              op.object(path)
	                  .writer()
	                  .write_bytes(content)
	                  .await
	                  .expect("create test data")
	          });
	  
	          Self {
	              op,
	              path: path.to_string(),
	          }
	      }
	  }
	  ```
	- 然后为这个结构体实现 `Drop`
		- ```rust
		  impl Drop for TempData {
		      fn drop(&mut self) {
		          TOKIO.block_on(async {
		              self.op
		                  .object(&self.path)
		                  .delete()
		                  .await
		                  .expect("cleanup test data");
		          })
		      }
		  }
		  ```
	- 需要注意的是
		- 如果在创建这个结构体的时候使用
			- ```rust
			  let _ = TempData::generate()
			  ```
		- 编译器会立刻执行 Drop，因为这个值已经不再使用了
		- 所以需要显式声明这边变量，然后以某种形式使用它，保证它的生命周期一直持续到测试结束
		- 我的做法是在最后手动调用 `std::mem::drop`
-
- 参考资料
	- [Trait std::ops::Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html)
	- [使用 Drop Trait 运行清理代码](https://kaisery.github.io/trpl-zh-cn/ch15-03-drop.html)