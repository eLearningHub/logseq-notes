type:: [[Product]]
features:: [[Version Control]]

- 将所有的 commit squash 成同一个
	- > 来自 [[Rust]] 社区： [Cargo’s crate index: upcoming squash into one commit](https://internals.rust-lang.org/t/cargos-crate-index-upcoming-squash-into-one-commit/8440)
	- ```shell
	  git reset $(git commit-tree HEAD^{tree} -m "Roll index into one commit")
	  ```
- Git 内部原理 - 传输协议
	- https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE
-