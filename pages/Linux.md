type:: [[Product]]
features:: [[OS]]

- 查看网卡型号
	- ```shell
	  lspci | grep Network
	  ```
- [[Cannot get MediaTek-based wifi to work]]
- [[配置 ssh-agent 服务]]
- [[配置 vmware 服务]]
-
- 避免服务器自动休眠
	- 朋友的 [[Archlinux]] 服务器总是会在 20 分钟后自动休眠，用起来非常不方便
	- 查看日志后看到
		- ```
		  Feb 09 18:08:02 arch systemd[1]: Reached target Sleep.
		  Feb 09 18:17:23 arch systemd[1]: Stopped target Sleep.
		  Feb 09 18:37:24 arch systemd[1]: Reached target Sleep.
		  Feb 09 18:45:07 arch systemd[1]: Stopped target Sleep.
		  Feb 09 19:05:08 arch systemd[1]: Reached target Sleep.
		  Feb 09 19:07:50 arch systemd[1]: Stopped target Sleep.
		  Feb 09 19:27:51 arch systemd[1]: Reached target Sleep.
		  Feb 09 19:27:58 arch systemd[1]: Stopped target Sleep.
		  ```
		- 看起来是 systemd 的自动休眠机制，采用简单粗暴的 `systemctl mask sleep.target` 解决了问题
-
- 修改 Caps 的映射
	- 可以使用 `localectl`，他会创建 `/etc/X11/xorg.conf.d/00-keyboard.conf` 配置文件
	- ```shell
	  localectl --no-convert set-x11-keymap cz,us pc104 "" ctrl:swapcaps
	  ```
	- 参考资料
		- [Xorg/Keyboard configuration](https://wiki.archlinux.org/title/Xorg/Keyboard_configuration#Swapping_Caps_Lock_with_Left_Control)