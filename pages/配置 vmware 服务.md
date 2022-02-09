- 安装
	- amd64 的平台可以直接安装，其他平台要参考 wiki 做调整
		- ```shell
		  yay -S vmware-workstation
		  ```
	- 安装完毕后需要重启来加载内核模块，也可以直接通过 [[modprobe]] 直接加载
		- ```shell
		  modprobe -a vmw_vmci vmmon
		  ```
	- vmware-workstation 已经打包好了 systemd service，可以直接启用
		- ```shell
		  systemd start vmware-networks.service --enable
		  systemd start vmware-usbarbitrator.service --enable
		  ```
- 技巧
	- 虚拟机能够在不同的 Host 之间直接共享
	- vmplayer 提供个人非商业的免费 license，workstation 则需要商业密钥
	- 确保直接内存访问
		- vmware 会把 guest 的内存写进 host 的文件当中，如果确认 host 内存足够的话，可以开启内存访问来降低磁盘 IO 开销，代价是内存占用提高
			- 修改对应的 vmx 文件
			- ```ini
			  MemTrimRate = "0"
			  sched.mem.pshare.enable = "FALSE"
			  prefvmx.useRecommendedLockedMemSize = "TRUE"
			  mainmem.backing = "swap"
			  ```
	- 直接挂载 NTFS 文件系统的话，会因为权限问题导致 vmware 无法正常启动虚拟机
		- 现象是 GUI 直接退出，无任何提示
		- 将对应的虚拟机目录 `chown` 为当前用户即可
- References
	- https://wiki.archlinux.org/title/VMware