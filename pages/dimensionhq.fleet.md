title:: dimensionhq/fleet

- https://github.com/dimensionhq/fleet
- https://fleet.rs
-
- 主要是结合了 [sccache](https://github.com/mozilla/sccache) 并做了一些编译参数的优化
  id:: 626512dd-3120-4b0a-bbee-ed71abc00604
-
- fleet 与 cargo 的对比
	- ```shell
	      Finished dev [unoptimized + debuginfo] target(s) in 3m 30s
	  cargo b  571.45s user 51.82s system 295% cpu 3:30.66 total
	  ```
	- ```shell
	  	Finished dev [unoptimized + debuginfo] target(s) in 2m 51s
	  fleet build  528.42s user 52.92s system 338% cpu 2:51.53 total
	  ```
	- 感觉有一些提升，但不是非常明显
-
- 不喜欢的地方
	- 默认的安装脚本不支持 archlinux
		- ```shell
		  curl -L get.fleet.rs | sh
		  ```
		- 尝试调用 apt install sccache = =
		- 有 rust 环境的同学可以使用下列命令安装
			- ```shell
			  cargo install fleet-rs
			  ```
	- 文档比较简陋，没有介绍自己是怎么做到的， 感觉非常黑箱
	- 默认生成的配置会覆盖全局的 cargo 设置，使得之前设置的 [[rui314/mold]] 没有正常工作