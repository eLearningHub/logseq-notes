- 日志
	- ```rust
	  2022-03-13T14:06:31.619308Z  INFO opendal::io: object 2/1/_b/660e9f382bec4fcd91e4c84ea0ede809.parquet poll_read: size 0
	  2022-03-13T14:06:31.619355Z  INFO opendal::io: object 2/1/_b/400bdec484a74a31a1c701084adda0e0.parquet poll_read: size 176
	  2022-03-13T14:06:31.619373Z  INFO opendal::io: object 2/1/_b/c1338300274e4e1291ca4f6196517bd1.parquet poll_read: size 63
	  2022-03-13T14:06:31.619392Z  INFO opendal::io: object 2/1/_b/ef85ab1e5d7b4838ae649398c3f828d9.parquet poll_read: size 63
	  2022-03-13T14:06:31.619415Z  INFO opendal::io: object 2/1/_b/367eac4c940b4fa09c25a1ecc1309ab4.parquet poll_read: size 63
	  2022-03-13T14:06:31.619456Z  INFO opendal::io: object 2/1/_b/381dc82e62f745de8e14fcaccd3a05e4.parquet poll_read: size 63
	  2022-03-13T14:06:31.619475Z ERROR databend_query::storages::fuse::io::block_reader: read file 2/1/_b/660e9f382bec4fcd91e4c84ea0ede809.parquet total 3783 at offset 1138 size 63: unexpected end of file
	  
	  ```