- [[Rust]] 多版本管理的组件
	- 其他语言中类似的工具包括 Ruby's rbenv, Python's pyenv, or Node's nvm.
-
- 安装
	- ```shell
	  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
	  ```
	- Windows 可以使用专门的安装程序: https://win.rustup.rs/x86_64
-
- toolchain file
	- ```toml
	  [toolchain]
	  channel = "nightly-2020-07-10"
	  components = [ "rustfmt", "rustc-dev" ]
	  targets = [ "wasm32-unknown-unknown", "thumbv2-none-eabi" ]
	  profile = "minimal"
	  ```
	- 可以在项目根目录增加配置文件 `rust-toolchain.toml` 来指定一组特定的 rust 版本
	- `rustc`，`cargo` 这些内置命令执行的时候，rustup 都会首先选择使用哪个 toolchain
		- 选择顺序参考 [Overrides](https://rust-lang.github.io/rustup/overrides.html#overrides)
	- 所以在 CI 等环境里面，实际上不需要做额外的 setup
-
- 工作原理
  collapsed:: false
	- `~/.cargo/bin` 下面的 `cargo` 和 `rustc` 等组件实际上是一个 proxy，会把对应的命令转发给具体的 toolchain
		- 这些 proxy 实际上都是同一个二进制，通过不同的 args[0] 来切换行为
		- ```shell
		  > md5sum ~/.cargo/bin/*
		  
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/cargo
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/cargo-clippy
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/cargo-fmt
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/cargo-miri
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/clippy-driver
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/rls
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/rustc
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/rustdoc
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/rustfmt
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/rust-gdb
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/rust-lldb
		  410d383b027062058a66f6549045be3f  /home/xuanwo/.cargo/bin/rustup
		  
		  ```
	- 实现 https://github.com/rust-lang/rustup/blob/master/src/bin/rustup-init.rs#L1-L12
		- ```rust
		  // A list of all binaries which Rustup will proxy.
		  pub static TOOLS: &[&str] = &[
		      "rustc",
		      "rustdoc",
		      "cargo",
		      "rust-lldb",
		      "rust-gdb",
		      "rust-gdbgui",
		      "rls",
		      "cargo-clippy",
		      "clippy-driver",
		      "cargo-miri",
		  ];
		  
		  ```
		- ```rust
		  		Some(n) => {
		              if TOOLS.iter().chain(DUP_TOOLS.iter()).any(|&name| name == n) {
		                  proxy_mode::main(n)
		              } else {
		                  Err(anyhow!(format!(
		                      "unknown proxy name: '{}'; valid proxy names are {}",
		                      n,
		                      TOOLS
		                          .iter()
		                          .chain(DUP_TOOLS.iter())
		                          .map(|s| format!("'{}'", s))
		                          .collect::<Vec<_>>()
		                          .join(", ")
		                  )))
		              }
		          }
		  ```
- 参考资料
	- [Rustup Introduction](https://rust-lang.github.io/rustup/index.html#introduction)