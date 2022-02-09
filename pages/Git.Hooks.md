title:: Git/Hooks

- 经常忘记在提交之前检查代码是不是 format 或者 clippy 检查过，可以配置一个 git hook
	- 根据自己的需求选择 hook 触发的时间，记得要 `chmod +x` 允许执行
		- 在 Commit 之前：`.git/hooks/pre-commit`
		- 在 Push 之前： `.git/hooks/pre-push`
	- ```bash
	  #!/bin/bash
	  
	  set -eu
	  
	  # Check fmt
	  if ! cargo fmt --all -q --check
	  then
	      cargo fmt --all
	      echo "Fmt not happy, please add again."
	      exit 1
	  fi
	  
	  # Check clippy
	  if ! cargo clippy -- -D warnings
	  then
	      echo "Clippy not happy."
	      exit 1
	  fi
	  ```