- RFC: [2930-read-buf](https://rust-lang.github.io/rfcs/2930-read-buf.html)
- Tracking Issue: https://github.com/rust-lang/rust/issues/78485
-
- Read trait 的函数签名是
	- ```rust
	  pub trait Read {
	      fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>;
	  }
	  ```
- 其中的 buf 要求我们必须先 init
	- ```rust
	  let mut buf = vec![0; 4096];
	  let nread = reader.read(&mut buf)?;
	  process_data(&buf[..nread]);
	  ```
- 这并不合理，因为 buf 马上就要被写入新的数据了，这个初始化完全是没有必要的，理论上来说我们应当可以这样
	- ```rust
	  let mut buf = Vec::with_capacity(4096);
	  unsafe { buf.set_len(4096); }
	  let nread = reader.read(&mut buf)?;
	  process_data(&buf[..nread]);
	  ```
- 但是这并不安全：`Read` 不是 unsafe 的，我们不能假定它的行为，用户可能会实现这样的 Read：
	- ```rust
	  struct BrokenReader;
	  
	  impl Read for BrokenReader {
	      fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
	          Ok(99999999999)
	      }
	  }
	  ```
		- 什么都不做，但是返回一个巨大的 read size，这可能会导致应用读取到错误的数据
	- ```rust
	  struct BrokenReader2;
	  
	  impl Read for BrokenReader2 {
	      fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
	          if buf[0] == 0 {
	              buf[0] = 1;
	          } else {
	              buf[0] = 2;
	          }
	          Ok(1)
	      }
	  }
	  ```
		- 在读取的时候修改 buf 中的值，导致每次读取的结果都不一样
- 这个 RFC 就旨在提升性能的同时避免在 safe rust 中触发未定义行为
	- #question 为什么引入 ReadBuf 能解决上面的这些问题呢？
		- `ReadBuf` 会明确定义 filled / unfilled 和 initialized / uninitialized
			- 访问 `unfilled` 空间中的内容是 unsafe 行为，安全性由用户自行保证
-
- 本 RFC 会加入一个新的 `ReadBuf` 结构体用来包装一个可能还没有被初始化的空间
	- ```rust
	  /// A wrapper around a byte buffer that is incrementally filled and initialized.
	  ///
	  /// This type is a sort of "double cursor". It tracks three regions in the buffer: a region at the beginning of the
	  /// buffer that has been logically filled with data, a region that has been initialized at some point but not yet
	  /// logically filled, and a region at the end that is fully uninitialized. The filled region is guaranteed to be a
	  /// subset of the initialized region.
	  ///
	  /// In summary, the contents of the buffer can be visualized as:
	  /// ```not_rust
	  /// [             capacity              ]
	  /// [ filled |         unfilled         ]
	  /// [    initialized    | uninitialized ]
	  /// ```
	  pub struct ReadBuf<'a> {
	      buf: &'a mut [MaybeUninit<u8>],
	      filled: usize,
	      initialized: usize,
	  }
	  ```
- 然后在 `Read` trait 中增加新的方法
	- ```rust
	  impl Read for SomeReader {
	      fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
	          let mut buf = ReadBuf::new(buf);
	          self.read_buf(&mut buf)?;
	          Ok(buf.filled().len())
	      }
	  
	      fn read_buf(&mut self, buf: &mut ReadBuf<'_>) -> io::Result<()> {
	          ...
	      }
	  }
	  ```
	- `read_buf` 和 `read` 互为默认实现，标准库中增加了一个专门的 lint 来确保一个 Reader
-
- 参考资料
	- [IO Buffer Initialization](https://paper.dropbox.com/doc/MvytTgjIOTNpJAS6Mvw38)
	- [UNINIT READ/WRITE](https://blog.yoshuawuyts.com/uninit-read-write/)