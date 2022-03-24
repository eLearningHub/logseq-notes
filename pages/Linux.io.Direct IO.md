title:: Linux/io/Direct IO

- 使用 `O_DIRECT` 打开的文件在读取时需要注意
  id:: 621b05d1-578c-4311-b8bd-c34ec4448123
	- > fd is attached to an object which is unsuitable for reading; or the file was opened with the O_DIRECT flag, and either the address specified in buf, the value specified in count, or the file offset is not suitably aligned.
	- buf 的内存地址，buf 的大小，文件的 offset 都需要对齐
		- 在 [[Rust]] 中，这意味着我们需要给结构体加上显式的 align 标记
			- ```rust
			  #[repr(align(4096))]
			  struct Aligned([u8; CHUNK_SIZE as usize]);
			  ```
	- 参考资料
		- https://man7.org/linux/man-pages/man2/read.2.html