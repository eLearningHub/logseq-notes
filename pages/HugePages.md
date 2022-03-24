title:: HugePages

- HugePages：大内存页
-
- 通常 [[Linux]] 使用的内存页大小为 4KB，这对需要占用大量内存的应用来说不友好，会导致 [[CPU]] 寻址开销很大。HugePage 通过增加单个页表的大小来降低页表映射的数量，从而降低这些开销。此外，HugePage 还能够锁定这部分内存，禁止操作系统对它进行交换和释放。
-
- > 更大的内存页能够减少内存中的页表层级，这不仅可以降低页表的内存占用，也能降低从虚拟内存到物理内存转换的性能损耗；
  > 更大的内存页意味着更高的缓存命中率，CPU 有更高的几率可以直接在 TLB（Translation lookaside buffer）中获取对应的物理地址；
  > 更大的内存页可以减少获取大内存的次数，使用 HugePages 每次可以获取 2MB 的内存，是 4KB 的默认页效率的 512 倍；
-
- 参考资料
	- [Debian Wiki: Hugepages](https://wiki.debian.org/Hugepages)
	- [为什么 HugePages 可以提升数据库性能](https://draveness.me/whys-the-design-linux-hugepages/)