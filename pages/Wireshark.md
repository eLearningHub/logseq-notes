- 包分析工具
-
- [[Archlinux]] 安装
	- ```shell
	  pacman -S wireshark-qt 
	  ```
	- 注意需要把自己加进 wireshard 组，然后修改 `/usr/bin/dumpcap` 的权限
		- ```shell
		  sudo chmod +x /usr/bin/dumpcap
		  ```
-
- 启动 wireshark 之后需要等待它把所有的网卡都加载出来
-
-