- 感觉可以借鉴 [[litestream]] & [[Rockset]] 的思路，设计一个将 embedb 传输到 s3 的服务 (in rust)？
-
- 可以先从比较简单的 kv-db 开始
	- 需要完整的事务能力，最好是自带 snapshot export 的支持
-
- [[Ourobox]] 的需求
	- 支持 replicate & restore sqlite
		- https://www.sqlite.org/c3ref/wal_checkpoint_v2.html
		-
-
- sled
	- https://docs.rs/sled/latest/sled/struct.Db.html#method.export
	- ```rust
	  pub fn export(
	      &self
	  ) -> Vec<(Vec<u8>, Vec<u8>, impl Iterator<Item = Vec<Vec<u8>>>)>
	  ```
	- ```rust
	  pub fn import(
	      &self,
	      export: Vec<(Vec<u8>, Vec<u8>, impl Iterator<Item = Vec<Vec<u8>>>)>
	  )
	  ```
	- 感觉有点怪啊，这是要自己想办法做序列化吗？
- rocksdb？
	- https://github.com/tikv/rust-rocksdb
- sqlite？
-
- Ideas
	- 如果对性能要求不高的话，能否每次都 export？
		- 感觉还是需要一个真正的 CDC 支持
		- [Change Data Capture in Embedded Databases](https://www.embeddedcomputing.com/technology/software-and-os/os-filesystems-libraries/change-data-capture-in-embedded-databases)
		- [Data Change Notification Callbacks](https://www.sqlite.org/c3ref/update_hook.html)
		- [Change Data Capture: What It Is and How to Use It](https://rockset.com/blog/change-data-capture-what-it-is-and-how-to-use-it/)