title:: Rust/Attributes/cfg_attr

- `cfg_attr` 用于制定条件满足时设置 attr: `#[cfg_attr(condition, attribute)]`
	- 当条件满足时： `#[cfg_attr(condition, attribute)]` => `#[attribute]`
	- 相反，则是一个 noop
-
- 以下列情形为例：
	- ```rust
	  #![cfg_attr(feature = "nightly", feature(core, std_misc))]
	  ```
	- 如果用户启用了 nightly feature，那用户就相当于设置了：`#![feature(core, std_misc)]`
-
- 参考资料
	- [The Rust Reference]
	- [Quick tip: the #[cfg_attr] attribute](https://chrismorgan.info/blog/rust-cfg_attr/)