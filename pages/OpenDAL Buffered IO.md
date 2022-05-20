- [[OpenDAL]] 可以考虑返回一个 buffered reader
-
- 如果 [[OpenDAL]] 支持了这个，能不能用来对接已有的压缩算法呢？
-
- 以 [[gzip]] 为例
	- https://docs.rs/flate2/latest/flate2/bufread/struct.GzDecoder.html
	- ```Rust
	  use std::io::prelude::*;
	  use std::io;
	  use flate2::bufread::GzDecoder;
	  
	  // Uncompresses a Gz Encoded vector of bytes and returns a string or error
	  // Here &[u8] implements BufRead
	  
	  fn decode_reader(bytes: Vec<u8>) -> io::Result<String> {
	     let mut gz = GzDecoder::new(&bytes[..]);
	     let mut s = String::new();
	     gz.read_to_string(&mut s)?;
	     Ok(s)
	  }
	  ```
	- 可以使用 `Decompress`： https://docs.rs/flate2/latest/flate2/struct.Decompress.html
-
- aysnc-compression 做法是自己维护了 header (只有 [[gzip]] 做了这样的处理)
	- ```rust
	  impl Header {
	      fn parse(input: &[u8; 10]) -> Result<Self> {
	          if input[0..3] != [0x1f, 0x8b, 0x08] {
	              return Err(Error::new(ErrorKind::InvalidData, "Invalid gzip header"));
	          }
	  
	          let flag = input[3];
	  
	          let flags = Flags {
	              ascii: (flag & 0b0000_0001) != 0,
	              crc: (flag & 0b0000_0010) != 0,
	              extra: (flag & 0b0000_0100) != 0,
	              filename: (flag & 0b0000_1000) != 0,
	              comment: (flag & 0b0001_0000) != 0,
	          };
	  
	          Ok(Header { flags })
	      }
	  }
	  ```
-
- 所以 decompress 这个函数需要维护一个 input/output buffer
	- 这两个 buffer 都是有状态的，保证只消费到有效的数据
- 所以 API 要怎么设计？怎么跟其他的用法保持同步呢？
	- 仍然暴露一个 AsyncRead？
	- 听起来需要一个 `AsyncBufRead<F>`
- ```rust
  pub trait AsyncBufRead: AsyncRead {
      fn poll_fill_buf(
          self: Pin<&mut Self>, 
          cx: &mut Context<'_>
      ) -> Poll<Result<&[u8]>>;
      fn consume(self: Pin<&mut Self>, amt: usize);
  }
  ```
-
- aysnc-compression 的做法
	- ```rust
	  #[derive(Debug, Default)]
	  pub struct PartialBuffer<B: AsRef<[u8]>> {
	      buffer: B,
	      index: usize,
	  }
	  
	  pub trait Decode {
	      /// Reinitializes this decoder ready to decode a new member/frame of data.
	      fn reinit(&mut self) -> Result<()>;
	  
	      /// Returns whether the end of the stream has been read
	      fn decode(
	          &mut self,
	          input: &mut PartialBuffer<impl AsRef<[u8]>>,
	          output: &mut PartialBuffer<impl AsRef<[u8]> + AsMut<[u8]>>,
	      ) -> Result<bool>;
	  
	      /// Returns whether the internal buffers are flushed
	      fn flush(&mut self, output: &mut PartialBuffer<impl AsRef<[u8]> + AsMut<[u8]>>)
	          -> Result<bool>;
	  
	      /// Returns whether the internal buffers are flushed
	      fn finish(
	          &mut self,
	          output: &mut PartialBuffer<impl AsRef<[u8]> + AsMut<[u8]>>,
	      ) -> Result<bool>;
	  }
	  ```
	- 在读取 IO 的时候从 Reader 中取出数据并进行同步的 decode
		- ```rust
		  State::Decoding => {
		    let input = ready!(this.reader.as_mut().poll_fill_buf(cx))?;
		    if input.is_empty() {
		      // Avoid attempting to reinitialise the decoder if the reader
		      // has returned EOF.
		      *this.multiple_members = false;
		      State::Flushing
		    } else {
		      let mut input = PartialBuffer::new(input);
		      let done = this.decoder.decode(&mut input, output)?;
		      let len = input.written().len();
		      this.reader.as_mut().consume(len);
		      if done {
		        State::Flushing
		      } else {
		        State::Decoding
		      }
		    }
		  }
		  ```
		- 感觉好像跟 opendal 没啥关系啊
-
- [[2022-05-20]]
	- 维护一个库，能够支持 async read / sync decompress
	- 然后 opendal 跟这个库做对接，支持 stage 的时候也这样做
		- stage 也是在调用 format 这一套逻辑
		- ```rust
		  async fn csv_source(
		    ctx: Arc<QueryContext>,
		    schema: DataSchemaRef,
		    stage_info: &UserStageInfo,
		    reader: BytesReader,
		  ) -> Result<Box<dyn Source>> {}
		  ```
		- opendal 需要提供一个新的 Reader，让用户可以
			- 去 fetch bytes，处理数据，并 fetch 新的数据
			- 这一套逻辑可以与之前的共存，用户如果不 case 这些开销的话，可以直接使用 async 的 decompress
				- 如果用户需要自己处理调度逻辑，可以使用 sync 的 compress
	- async fill_buf ?
	- 然后提供一个同步的 advance
		- advance 本来就是同步的
	- 好像不需要额外做什么，只需要封装一下，变成 AsyncBuf
	- 但是这样的话 object 遇到一些组合问题
		- range_buf_reader?
	- 感觉直接暴露成 object 的 API 不太合适
	- 当作 trait 处理？
		- into_decompress_reader()
		- into_buf_reader()
		- 这个感觉不错
	- 然后就是考虑一下怎么用
		- 可以直接调用 into_decompress_reader，读取被解压缩的数据
		- 也可以调用 into_buf_reader()，手动解压
			- fill_buffer().await 拿到 buffer 调用 decompress
		- 还需要自己封装吗？
	-