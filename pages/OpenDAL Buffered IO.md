- [[OpenDAL]] 可以考虑返回一个 buffered reader
-
- fill_buf?
- get_buf?
-
- 如果 [[OpenDAL]] 支持了这个，能不能用来对接已有的压缩算法呢？
-
- 以 [[gzip]] 为例
	- https://docs.rs/flate2/latest/flate2/bufread/struct.GzDecoder.html
	- ```[[Rust]]
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
-
- aysnc-compression 做法是自己维护了 header (只有 gzip 做了这样的处理)
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