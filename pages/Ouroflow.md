- 感觉可以借鉴 [[litestream]] & [[Rockset]] 的思路，设计一个将 embedb 传输到 s3 的服务 (in rust)？
-
- 可以先从比较简单的 kv-db 开始
	- 需要完整的事务能力，最好是自带 snapshot export 的支持
-
- [[Ourobox]] 的需求
	- 支持 replicate & restore sqlite
		- https://www.sqlite.org/c3ref/wal_checkpoint_v2.html
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
	- export 会锁住整个表
- rocksdb？
	- https://github.com/tikv/rust-rocksdb
- sqlite？
	- 或许可以使用 Online Backup？
- Ideas
	- 如果对性能要求不高的话，能否每次都 export？
		- 感觉还是需要一个真正的 CDC 支持
		- [Change Data Capture in Embedded Databases](https://www.embeddedcomputing.com/technology/software-and-os/os-filesystems-libraries/change-data-capture-in-embedded-databases)
		- [Data Change Notification Callbacks](https://www.sqlite.org/c3ref/update_hook.html)
		- [Change Data Capture: What It Is and How to Use It](https://rockset.com/blog/change-data-capture-what-it-is-and-how-to-use-it/)
	- Rust Sqlite [rusqlite](https://docs.rs/rusqlite/latest/rusqlite/struct.Connection.html)
		- > A Backup handle exposes three methods: step will attempt to back up a specified number of pages, progress gets the current progress of the backup as of the last call to step, and run_to_completion will attempt to back up the entire source database, allowing you to specify how many pages are backed up at a time and how long the thread should sleep between chunks of pages.
	- [Using the SQLite Online Backup API](https://www.sqlite.org/backup.html)
		- 传统方法需要锁全表
		- >  The online backup API allows the contents of one database to be copied into another database file, replacing any original contents of the target database.
		- > The copy operation may be done incrementally, in which case the source database does not need to be locked for the duration of the copy, only for the brief periods of time when it is actually being read from.
		- > This allows other database users to continue without excessive delays while a backup of an online database is made.
		- > The effect of completing the backup call sequence is to make the destination a bit-wise identical copy of the source database as it was when the copying commenced. (The destination becomes a "snapshot.")
		- 新的方法不要求锁全表，取而代之的是增量式的备份
			- 每次复制 X pages，然后 sleep 一定实现，然后再继续复制
		- 如果 sleep 过程中有了新的写入，backup 会重新开始
			- 最极端的情况下，这个 backup 任务可能永远也无法完成？。。
			- 还是要看情况
				- 如果是不同的线程修改了，那 backup 会自动重新开始
				- 如果是同一个线程，backup 会自动同步这个变更
		- 这要求用户必须使用跟 ouroflow 同一个 connection，实际上不太可能
		-
		-
	- 或者我们可以先不考虑那么多，先朴素的把整个