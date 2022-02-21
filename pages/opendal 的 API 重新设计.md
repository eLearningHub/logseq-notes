- 返回一个 Object，然后 Object 提供操作的支持
- 比如说 reader() 和 seekable_reader()
- 支持两种操作，一种是 read，另一种是 cursor
	- 是不是从底层就得返回两个参数
- 区分 Stream 和 Reader
	- Convert stream to async reader
		- https://api.rocket.rs/master/src/rocket/response/stream/reader.rs.html#13-96
- Reader vs StatefulReader
- 在 Object 中维护更多状态？
- 默认返回的 Reader 就是 Seekable 的会不会引起混淆？
	- Stream 其实有点像默认实现了 read_buf?
- 如果统一返回 reader 的话，如何保持 read_all 的性能不发生回退呢？
	- Sequential Reading
	- Disorder Reading
- 每次 open 的时候主动 stat 一下？
-
- 该给用户暴露怎样的接口？
-
- 如何避免重复的 stat？
	- 支持外部指定 size_hint()
- ```rust
  let o = op.object("xxxx");
  let () = o.delete().await?;
  let meta = o.stat().await?;
  
  let r = o.new_reader().read().await?;
  let w = o.new_writer();
  ```
- ```rust
  let o = op.objects(path)
  ```
-
- 有没有办法统一成一个 Reader 呢？
	- new_reader() -> Reader
	- r.read()
- 要考虑一下对用户和对 service 的 API
-
- 对用户暴露一个 new_reader 和 new_writer
	- Reader 会使用如下 API
		- acc.random_read()
	- Writer 会使用如下 API (large file support？)
		- acc.write()
		-
-
- service 必须实现 sequential_read 和 random_read 其中一个，然后两个互为默认实现?