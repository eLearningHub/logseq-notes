- sqllogictest 是 [[SQLite]] 社区推出的一种 sql 测试格式
-
- 看起来大概是这样
	- ```test
	  query I
	  select * from example_basic
	  ----
	  Alice
	  Bob
	  Eve
	  
	  statement ok
	  drop table t
	  ```
-
- 支持执行 SQL 语句，比如
	- ```test
	  statement ok
	  drop table t
	  ```
	- 表示预期 `drop table t` 会返回 ok
- 支持测试 query 并比对返回的结果
	- ```test
	  query <type-string> <sort-mode> <label>
	  ----
	  <expected result>
	  ```
	- `type-string` 指定返回的结果集
		- `T` 文本，`I` 整形，`R` 浮点
		- 比如 `III` 就表示预期每一列返回三个整数
			- ```test
			  query III rowsort
			  select * from example_rowsort
			  ----
			  1 10 2333
			  10 100 2333
			  2 20 2333
			  ```
	- `sort-mode` 支持对结果做排序
	- `lable` 支持对结果打标签，后面可以重用
- 支持通过 `include` 导致别的测试内容，可以避免每次都重复
-
- 参考实现： https://www.sqlite.org/sqllogictest/doc/trunk/about.wiki
- Rust 实现：[sqllogictest-rs](https://github.com/singularity-data/sqllogictest-rs)