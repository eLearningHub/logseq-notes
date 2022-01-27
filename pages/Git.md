type:: [[Product]]
features:: [[Version Control]]

- 将所有的 commit squash 成同一个
	- > 来自 [[Rust]] 社区： [Cargo’s crate index: upcoming squash into one commit](https://internals.rust-lang.org/t/cargos-crate-index-upcoming-squash-into-one-commit/8440)
	- ```shell
	  git reset $(git commit-tree HEAD^{tree} -m "Roll index into one commit")
	  ```