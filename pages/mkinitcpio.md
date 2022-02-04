- [[Archlinux]] 每次更新内核的时候都会出现类似的提示
- ```text
  ==> WARNING: Possibly missing firmware for module: aic94xx
  ==> WARNING: Possibly missing firmware for module: bfa
  ==> WARNING: Possibly missing firmware for module: qed
  ==> WARNING: Possibly missing firmware for module: qla1280
  ==> WARNING: Possibly missing firmware for module: qla2xxx
  ==> WARNING: Possibly missing firmware for module: wd719x
  ==> WARNING: Possibly missing firmware for module: xhci_pci
  ```
- 根本原因是 `linux-firmware` 中没有预置这些固件
- Archwiki 中给出的[解决方案](https://wiki.archlinux.org/title/Mkinitcpio#Possibly_missing_firmware_for_module_XXXX) 是安装需要的固件，比如
	- > For aic94xx, install aic94xx-firmware
	- > For wd719x, install wd719x-firmware
	- > For xhci_pci, install upd72020x-fw
- 但问题在于，我们并用不到这些模块，所以更好的方案是在 mkinitcpio 的时候把特定的模块去掉
	- 这需要修改 `initcpio` 的配置文件，可以参考 [此处](https://gist.github.com/imrvelj/c65cd5ca7f5505a65e59204f5a3f7a6d?permalink_comment_id=3827257#gistcomment-3827257)