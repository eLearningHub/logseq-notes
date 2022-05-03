- 必要的准备
	- Clone [[Rust]]
		- ```shell
		  git clone https://github.com/rust-lang/rust.git
		  cd rust
		  git submodule update --init --recursive
		  ```
	- 使用 `x.py` 配置环境
		- ```shell
		  ./x.py setup
		  ```
		- 这里会自动的进行一些交互式的配置，参考 [Create a config.toml](https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html#create-a-configtoml)
		- 如果主要是做一些 lib 相关的开发(比如说标准库)，可以选择 `library` 即可
- 常用命令
	- Build: `./x.py build`
	- Check: `./x.py check`
	- Test: `./x.py test`
- 常用技巧
	- Rust 测试量很大，最好只测试跟自己相关的
		- ```shell
		  # test a crate
		  ./x.py test --stage 1 library/alloc
		  # test a mod
		  ./x.py test --stage 1 library/alloc --test-args sync
		  ```
	- 通过 `--stage 0|1|2` 来指定不同的 build stage
		- 标准库开发比较常用 `--stage 1`
		- 比如 `./x.py build --stage 1`
- 如何 Stable 一个 lib Feature？
	- 在开发功能的时候需要增加 unstable 的标注
		- `#[unstable(feature = "foo", issue = "1234", reason = "lorem ipsum")]`
		- 这样用户想要使用这个 feature 就必须显式导入： `#![feature]`
	- 当这个feature 稳定时，这个标注会被修改为 `stable`
		- `#[stable(feature = "foo", since = "1.420.69")]`
	- 具体流程如下
		- ping @T-libs-api 的成员来开启一个 FCP (Final commenting period) 流程
			- 在此期间 libs 团队的成员会对这个 feature 做最后的 review
		- 将代码中所有的 `#[unstable(...)]` 修改为 `#[stable(since = "version")]`
			- version 必须是当前的 nightly
		- 删除 test / doc-test 中的 `#![feature(...)]`
			- 因为不再需要这个 feature gate 了
		- 将这些所有的变更提交为一个 PR 并链接到对应的 Tracking issue
	- 参考: [Stability attributes](https://rustc-dev-guide.rust-lang.org/stability.html#stabilizing-a-library-feature)
- 常用链接
	- [Guide to Rustc Development](https://rustc-dev-guide.rust-lang.org/getting-started.html)
	- [Rust Forge](https://forge.rust-lang.org/)