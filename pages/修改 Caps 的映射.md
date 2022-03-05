- 可以使用 `localectl`，他会创建 `/etc/X11/xorg.conf.d/00-keyboard.conf` 配置文件
- ```shell
  localectl --no-convert set-x11-keymap cz,us pc104 "" ctrl:swapcaps
  ```
- 参考资料
	- [Xorg/Keyboard configuration](https://wiki.archlinux.org/title/Xorg/Keyboard_configuration#Swapping_Caps_Lock_with_Left_Control)