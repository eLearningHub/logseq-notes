title:: Iteration/4
type:: Iteration
date:: 2022-01-03 - 2022-01-16

- [Done](https://github.com/users/Xuanwo/projects/2/views/1?filterQuery=iteration%3A%22Iteration+4%22)
-
- DAL2 的重构
	- 实现了 s3 的支持，单元测试
	- callback 重构
- Rust
	- 为 PathBuf 实现了 try_reserve 和 try_reserve_exact，等待 review
- logseq-publish 的改进
	- 修了一个小 bug
	- 重构了 github actions
- TiDB 黑客马拉松
	- Interstellar -> TiDB 冷热数据分离
	- 通过分区表将冷数据迁移到 s3 上，然后使用 [[AWS Athena]] 来读取 s3 上的数据
	- 可惜没拿大奖，我们的想法跟一等奖撞车了
		- 具体的差异是一等奖没有使用 [[AWS Athena]] 而是直接使用 [[S3/Select]]
		- 在功能上他们可以完整的跑完 [[TPC-H]] 测试，我们只能跑个别几个，所以就被 PK 下去了
		- Paper [[PushdownDB: Accelerating a DBMS using S3 Computation]] 也是这个思路
	- 我在这个项目里面的工作是为 [[TiDB]] dumpling 实现了 [[Parquet]] 支持
-