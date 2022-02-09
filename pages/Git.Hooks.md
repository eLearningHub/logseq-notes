title:: Git/Hooks

- 经常忘记在提交之前检查代码是不是 format 或者 clippy 检查过，可以配置一个 `pre-push` hook
	- ```bash
	  #!/bin/bash
	  
	  set -eu
	  
	  # Check fmt
	  if ! cargo fmt -- --check
	  then
	      cargo fmt
	      echo "Fmt not happy, please add again."
	      exit 1
	  fi
	  
	  # Check clippy
	  if ! cargo clippy --all-targets -- -D warnings
	  then
	      echo "Clippy not happy."
	      exit 1
	  fi
	  ```