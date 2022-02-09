- https://forums.gentoo.org/viewtopic-p-8683048.html?sid=8f355c65d71dfea5e80b5d9041abd0b4
- 同事买了一个新电脑，自带的网卡是 MediaTek 0608
- ```
  04:00.0 Network controller: MEDIATEK Corp. Device 0608
  	Subsystem: MEDIATEK Corp. Device 0608
  	Flags: fast devsel, IRQ 15, IOMMU group 22
  	Memory at e2100000 (64-bit, prefetchable) [disabled] [size=1M]
  	Memory at e2200000 (64-bit, prefetchable) [disabled] [size=16K]
  	Memory at e2204000 (64-bit, prefetchable) [disabled] [size=4K]
  	Capabilities: <access denied>
  ```
- 先后尝试了 ubuntu 18.04，21.04，系统自带的驱动都没法驱动
- 用类似的型号去搜索，在 Gentoo 论坛找到了答案
- 这个问题是[这个 Patch](https://patchwork.kernel.org/project/linux-wireless/patch/84ab45bf42f57fd0301c156ffc11d0fe330ff1f8.1636857817.git.deren.wu@mediatek.com/)修复的，合并于 2021-11，太新了。。
- 折腾了半天 ubuntu 下升级内核无果，最后决定上 archlinux (