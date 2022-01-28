type:: [[Iteration]]
date:: 2022-01-17 - 2022-01-30

- [Done](https://github.com/users/Xuanwo/projects/2/views/1?filterQuery=iteration%3A%22Iteration+5%22)
-
- Rust
	- PR [std: Implement try_reserve and try_reserve_exact on PathBuf](https://github.com/rust-lang/rust/pull/92513) 已经成功 merge 了，将会在 1.60 中 release
	- 至此 feature `try_reserve_2` 需要的工作已经全部完成，接下来就是推进 Stabilization 的事宜了
- singularity-data/sqllogictest-rs
	- Sqllogictest-rs 是 singularity-data 团队出品的 SQL 正确性验证框架，基于业界现成的 [[sqllogictest]] 格式实现。
	- 由于这个 Iteration 看了一些关于测试的内容，于是顺手贡献了一些东西
	- [parser: Logic cleanup around sql result check](https://github.com/singularity-data/sqllogictest-rs/pull/14) 对判断逻辑做了一些小优化，水到了第一个贡献
	- [runner: Implement validator support](https://github.com/singularity-data/sqllogictest-rs/pull/15) 就相对更有意思一些了。
		- sqllogictest-rs 之前只支持静态的内容，没法验证像
- Xuanwo
	- 这个 Iteration 突击学了不少跟 Async Rust 相关的概念和知识，包括
		- Pin: [[Rust/std/Pin]]
		- Futures: [[Rust/std/Future]]
		- Send & Sync: [[Rust/std/Send]]
	- 感觉收获不少，从这个 Iteration 开始我试着在 Twitter 每天分享今天学到了什么。
	- 最让我开心的是 关于 Futures 的分享甚至还启发了原作者，帮助他把自己的文章补充更加完整。形成了一个完整的正向循环，这种互相连接的感觉真的很棒。
		- ![image.png](../assets/image_1643361629991_0.png)
-
- {{query (between [[2022-01-17]] [[2022-01-30]])}}