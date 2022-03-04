title:: Rust/Cargo/Lock

- [Why do binaries have Cargo.lock in version control, but not libraries?](https://doc.rust-lang.org/cargo/faq.html#why-do-binaries-have-cargolock-in-version-control-but-not-libraries)
-
- `Cargo.lock` 的设计目标是可重现(确定性)构建
- 对应用来说维护 Cargo.lock 能避免构建的时候使用的依赖跟自己预期的不一致
	- 一方面能减少调试和排错的成本
	- 另一方面能避免供应链攻击
- 而对库来说，维护 Cargo.lock 没有用处
	- 首先现在的 Cargo 实现不会去检查每个库的 lock 文件
	- 其次每个库只能指定自己的依赖，对全局的情况不了解
		- 如果构建的时候使用所有的 lock 文件会导致多个版本的库参与构建，更严重的情况下会出现版本冲突
- 总的来说，只有应用(二进制)才了解和应该决定最终产物的依赖关系是怎么样的，所以二进制需要 Cargo.lock 而库不需要
-