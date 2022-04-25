- 纯过程式的包管理工具
-
- 一般说的 Nix 通常有三样东西
	- Nix 包管理工具
	- Nix 语言
	- NixOS
-
- 在其他发行版中也可以使用 Nix 来管理依赖
-
- 比如说在 archliunx 上可以使用
	- ```shel
	  pacman -S nix
	  ```
	- 需要把当前用户加进 nix-users
		- ```shell
		  sudo usermod -a -G nix-users xuanwo
		  ```
		- 否则没有权限访问 nix socket，跟 dockerd 的行为有点像
	- 然后启用服务即可
		- ```shell
		  systemd start nix-daemon
		  ```
-
- 在正式使用之前还需要做一些简单配置
	- 启用 unstable 源
		- ```shell
		  nix-channel --add https://nixos.org/channels/nixpkgs-unstable
		  nix-channel --update
		  ```
	- 将 profile bin 加入 PATH
		- ```shell
		  export PATH="$HOME/.nix-profile/bin:$PATH"
		  ```
-
- 以使用 hello 为例
	- 安装 hello
		- ```shel
		  nix-env -iA nixpkgs.hello
		  ```
	- 卸载 hello
		- ```shell
		  nix-env --uninstall hello
		  ```
-
- 参考资料
	- Archwiki: https://wiki.archlinux.org/title/Nix