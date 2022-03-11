type:: [[Product]]
features:: [[Network]]

- tcpdump 是 [[Linux]] 平台下的网络数据采集分析工具。
-
- 监听特定网卡
	- ```shell
	  tcpdump -i wlan0
	  ```
- 监听来自主机 `192.168.0.11` 在端口 `280 上的
	- ```shell
	  tcpdump port 3000
	  ```
-
- 参考资料
	- [tcpdump(8) - Linux man page](https://linux.die.net/man/8/tcpdump)