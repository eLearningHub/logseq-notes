- cargo 是 [[Rust]] 的官方构建工具，通常使用 [[Rustup]] 来安装
-
- 配置
	- 跟 [[Git]] 一样，可以给常用的命令配置别名
		- ```toml
		  b = "build"
		  c = "check"
		  cc = "clippy --tests -- -D warnings"
		  t = "test"
		  ```
	- 可以修改编译中用到的参数，比如使用 [[rui314/mold]]
		- ```toml
		  [target.x86_64-unknown-linux-gnu]
		  linker = "clang"
		  rustflags = ["-C", "link-arg=-fuse-ld=/usr/bin/mold"]
		  ```
-
- 常用功能
	- build
		- `cargo build`: 构建整个项目
	- fmt
		- `cargo fmt`： 格式化整个项目
	- check
		- `cargo check`: 检查项目中的错误
		- 可以使用 `cargo fix` 来修复这些问题，最好不要无脑 fix
			- unused_import 这种去掉就行
			- unused_value 就得看看了，有时候可能是代码中漏了
	- clippy
		- `cargo clippy`: 比 check 更严格的检查，有的时候不太智能
		- 通常会这样用： `cargo clippy --tests -- -D warnings`
			- `--tests`: 默认不对 tests 做 clippy，加上以覆盖测试代码
			- `-D warnings`: 遇到 warnings 时返回 exit 1 使得脚本报错，CI 里面会需要
		- 跟 check 类似，提供了 `cargo clippy --fix` 来自自动修复
	- test
		- 运行指定测试: `cargo test -- "metrics::test_metric_server"`
		- 输出测试中打印到 stdout 的内容: `cargo test -- "metrics::test_metric_server" --show-output`
		- 显示完整的 Backtrace: `RUST_BACKTRACE=1 cargo test`
-
- 方便的插件
	- [killercup/cargo-edit](https://github.com/killercup/cargo-edit)
		- A utility for managing cargo dependencies from the command line.
		- 提供 `cargo add`, `cargo rm`, `cargo update` 等命令
-
- 参考资料
	- [The Cargo Book](https://doc.rust-lang.org/cargo/)
	-