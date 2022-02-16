- imagemagick 提供了 `convert` 命令，可以用来压缩图片
-
- 比如压缩一个 jpg 的尺寸为原来的一半
	- ```shell
	  convert -filter Cubic -resize 50% old.jpg new.jpg
	  ```