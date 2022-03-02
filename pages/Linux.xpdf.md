- xpdf 是 Linux 平台上一个免费的 PDF 阅读器和工具集，提供了一系列的有用工具，比如文字提取，图片转换，HTML 转换等等。
-
- 安装
	- ```shell
	  pacman -S xpdf
	  ```
-
- 提供如下工具
	- pdftotext
	- pdftops
	- pdftoppm
	- pdftopng
	- pdftohtml
	- pdfinfo
	- pdfimages
	- pdffonts
	- pdfdetach
-
- 将 pdf 转换为 png
	- ```shell
	  pdftopng xxx.pdf xxx
	  ```
	- 将会得到 `xxx-0000001.png`