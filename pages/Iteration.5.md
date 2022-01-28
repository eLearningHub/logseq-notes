type:: [[Iteration]]
date:: 2022-01-17 - 2022-01-30

- [Done](https://github.com/users/Xuanwo/projects/2/views/1?filterQuery=iteration%3A%22Iteration+5%22)
-
- 周期总结已经发布于 https://xuanwo.io/reports/2022-04/
-
- Rust
	- PR [std: Implement try_reserve and try_reserve_exact on PathBuf](https://github.com/rust-lang/rust/pull/92513) 已经成功 merge 了，将会在 1.60 中 release
	- 至此 feature `try_reserve_2` 需要的工作已经全部完成，接下来就是推进 Stabilization 的事宜了
- singularity-data/sqllogictest-rs
	- Sqllogictest-rs 是 singularity-data 团队出品的 SQL 正确性验证框架，基于业界现成的 [[sqllogictest]] 格式实现。
	- 由于这个 Iteration 看了一些关于测试的内容，于是顺手贡献了一些东西
	- [parser: Logic cleanup around sql result check](https://github.com/singularity-data/sqllogictest-rs/pull/14) 对判断逻辑做了一些小优化，水到了第一个贡献
	- [runner: Implement validator support](https://github.com/singularity-data/sqllogictest-rs/pull/15) 就相对更有意思一些了。
		- sqllogictest-rs 之前只支持静态的内容，没法验证会产生动态内容的 SQL 是否正确，比如说
			- ```SQL
			  SELECT now();
			  ```
			- ```SQL
			  SELECT * from system.tables where name = 'tables';
			  ```
		- [[Databend]] 目前的做法是在 Result 之外，再提供一个 `result_filter` 文件来过滤，需要用到 `sed`
			- 看起来大概是这样
			- result
				- ```text
				  system	tables	SystemTables	yyyy-mm-dd HH:MM:SS.sss +0000
				  ```
			- result_filter
				- ```text
				  \d\d\d\d-\d\d-\d\d \d\d:\d\d:\d\d[.]\d\d\d [+-]\d\d\d\d
				  yyyy-mm-dd HH:MM:SS.sss +0000
				  ```
		- 这样不仅实现和维护起来麻烦，测试的效率也不高
		- 所以我在这个 PR 中增加了 Validator 的抽象，让用户可以自己传入需要的 Validator，这样就能自行控制验证的逻辑了。
		- 用起来的感觉大概是这样：
			- ```rust
			  fn main() {
			      let script = std::fs::read_to_string(Path::new("examples/validator.slt")).unwrap();
			      let mut tester = sqllogictest::Runner::new(FakeDB);
			      // Validator will always return true.
			      tester.with_validator(|_, _| true);
			      tester.run_script(&script).unwrap();
			  }
			  ```
			- 通过 `with_validator` 传入一个 `Validator = fn(&Vec<String>, &Vec<String>) -> bool` 即可
		- 未来会尝试将部分 Databend 的测试迁移到 sqllogictest-rs 上来，这样我们的测试就是 pure rust，不再依赖外部的各种工具。
- bentoml/Yatai
	- Yatai 是 [&yetone](https://github.com/yetone) 主导设计和开发的 ML Ops 平台，乘着宣布开源的东风，我借机水了一个 PR，把所有的 Go import 都按照统一的格式进行了整理。期间还暴露了项目的 CICD 中 golangci-lint 没有正确运行的问题，这些八卦欢迎来 [*: Format import](https://github.com/bentoml/Yatai/pull/142) 查看。
- Databend
	- 剩下的工作就是跟 Databend 相关的了。
	- 目前主要关心的还是 Databend 的 Data Access Layer，它会负责 Databend 与持久化数据存储的交互。
		- 这个 Iteration 工作进展不少，DAL2 的接口基本成型，部分模块切换成了 DAL2，已有的测试用例全部通过。
		- 得益于 DAL2 的优秀抽象，我们能够增加一个通用的 Interceptor / Layer 层来运行用户对 DAL2 的行为做一些装饰，增加回调的处理。 这样就能进一步满足业务对 DAL2 的要求，比如说 Metric 等等。
		- 等到所有的模块都切换成 DAL2 之后，就能让社区一起参与进来，去实现更多的存储后端。
	- 除了 DAL 之外，我还做一些简单的工程效率工作，处理了一些跟 CI 相关的问题，包括为 Databend 引入了 peotry 来管理 Python 依赖，修复 CI 使用错误的 Rust 版本等。
		- CI 使用错误的 CI 版本是个很有意思的问题
		- 我们经常会使用这样的 action 来初始化 Rust 构建环境：
		- ```toml
		  - uses: actions-rs/toolchain@v1
		    with:
		    toolchain: stable
		  - uses: actions-rs/cargo@v1
		    with:
		    command: build
		    args: --release --all-features
		  ```
		- [[rustup]] 的实现决定了 cargo 在第一次运行的时候会自动选择合适的 toolchain，并不需要额外的 action 来指定。所以直接将 toolchain 这一步去掉就行
- Xuanwo
	- 这个 Iteration 突击学了不少跟 Async Rust 相关的概念和知识，包括
		- Pin: [[Rust/std/Pin]]
		- Futures: [[Rust/std/Future]]
		- Send & Sync: [[Rust/std/Send]]
	- 感觉收获不少，从这个 Iteration 开始我试着在 Twitter 每天分享今天学到了什么。
	- 最让我开心的是 关于 Futures 的分享甚至还启发了原作者，帮助他把自己的文章补充更加完整。形成了一个完整的正向循环，这种互相连接的感觉真的很棒。
		- ![image.png](../assets/image_1643361629991_0.png)
- 总的来说这个 Iteration 过的比较充实，我们年后再见~
-
- {{query (between [[2022-01-17]] [[2022-01-30]])}}