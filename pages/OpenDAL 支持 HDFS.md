- 可行的几个选择
	- webhdfs
	- [hdfs-rs](https://github.com/hyunsik/hdfs-rs)
		- libhdfs binding library
		- 而 libhdfs 是 hadoop 提供的 C API (基于 JNI 实现)
	- libhdfs
		- https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/LibHdfs.html
	- libhdfs3
		- https://github.com/ClickHouse/libhdfs3
			- > libhdfs3 designed as an alternative implementation of libhdfs, is implemented based on native Hadoop RPC protocol and HDFS data transfer protocol.
			- 使用了 hadoop 的 RPC 接口
			- 但是项目的维护质量也一般
		- DataFusion 选择了 wrap libhdfs3
			- https://github.com/datafusion-contrib/datafusion-hdfs-native
	- JNI
		- https://docs.rs/jni/latest/jni/
		- https://github.com/giovanniberti/robusta
		- https://github.com/astonbitecode/j4rs
		- jni 感觉好麻烦了，翻了半天文档，没搞懂怎么搞
-
- 关于 HDFS
	- https://hadoop.apache.org/docs/current/api/org/apache/hadoop/fs/FileContext.html
	- > The Hadoop file system supports a URI namespace and URI names. This enables multiple types of file systems to be referenced using fully-qualified URIs. Two common Hadoop file system implementations are
		- > the local file system: file:///path
		- > the HDFS file system: hdfs://nnAddress:nnPort/path
-
- 需要构建一个 hdfs 的库
	- 支持用户的多种环境
		- 2.10 -> 3.3
	- 问题是需要怎么做？感觉要有一个自动检测 lib 版本的机制？
	- hdfs-sys 需要通过 feature 来开关不同的 mod？
		- 然后 hdrs 的在编译时候检查 so 的情况？
	- 这套东西怎么搞啊- -
	- 还是说 hdfs-sys 在编译的时候就需要检查 so？
		-
- https://github.com/rust-lang/git2-rs/blob/master/libgit2-sys/build.rs
-
- 哇，终于成功的进行了一次构建
	- ```shell
	  HADOOP_HOME=/tmp/hadoop-3.2.3/hadoop-3.2.3 LD_LIBRARY_PATH=$HADOOP_HOME/lib/native:${JAVA_HOME}/lib/server cargo test
	  ```
	- ```rust
	      #[test]
	      fn test_x() {
	          unsafe {
	              hdfs_3_2_3::hdfsNewBuilder();
	          }
	      }
	  ```
- LD_LIBRARY_PATH 影响什么？
	- https://prefetch.net/articles/linkers.badldlibrary.html
	-
- ```shell
  HADOOP_HOME=/tmp/hadoop-3.2.3/hadoop-3.2.3 LD_LIBRARY_PATH=${HADOOP_HOME}/lib/native:${JAVA_HOME}/lib/server CLASSPATH=${HADOOP_HOME}/share/hadoop/common/*:${HADOOP_HOME}/share/hadoop/common/lib/*:${HADOOP_HOME}/share/hadoop/hdfs/*:${HADOOP_HOME}/share/hadoop/hdfs/lib/*:${HADOOP_HOME}/etc/hadoop/* cargo test test_x
  ```