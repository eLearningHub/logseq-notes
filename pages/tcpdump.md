type:: [[Product]]
features:: [[Network]]

- tcpdump 是 [[Linux]] 平台下的网络数据采集分析工具
	- tcpdump 基于 [libpcap](https://www.tcpdump.org/manpages/pcap.3pcap.html) 开发，有更高级的需求可以使用 libpcap 来自定义
-
- 监听特定网卡
	- ```shell
	  tcpdump -i wlan0
	  ```
- 监听来自主机 `192.168.0.11` 在端口 `80` 上的 TCP 包
	- ```shell
	  tcpdump tcp port 80 and src host 192.168.0.11
	  ```
-
- 跟 [[Wireshark]] 可以搭配使用
	- 通过指定 `-w` 参数可以将抓包数据保存下来
		- ```shell
		  tcpdump tcp port 80 and src host 192.168.0.11 -w ./http.pcap
		  ```
	- 然后在 wireshark 中直接打开即可
-
- [[kubernetes]] 环境下可以使用 [[kubectl]] 的插件 [ksniff](https://github.com/eldadru/ksniff)
	- 它集成了 tcpdump 和 [[wireshark]] ，能够方便的查看指定 pod 的流量
-
- 参考资料
	- [tcpdump(8) - Linux man page](https://linux.die.net/man/8/tcpdump)