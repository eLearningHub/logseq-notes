title:: Data Layers: I have a dream

-
- 状态
	- 已经输出成内部的分享， [[2022-01-14]] 整理后发表博客
-
- 期望
	- 具体的语法待定
	- ```sql
	  -- Create Stage
	  create stage test
	        type = S3
	        credential = 'hmac:ak:sk'
	        endpoint = 'https://s3.awsamazon.com'
	        bucket = 'bucket_name'
	        region = 'us-west-1'
	        root = '/path/to/data';
	  ```
	- ```sql
	  -- Copy data from stage into table
	  copy into test_csv from '@test/path/to/data'
	        format CSV
	        record_delimitor = ',';
	        
	  -- Copy data from table to stage
	  copy into '@test/path/to/data' from test_csv;
	  ```
- 现状
	- storage  仅支持 s3，azblob 和 local fs
	- storage 不支持服务器端加密
	- storage 接口不正交，难以扩展新的存储支持
	- format 仅支持 parquet / csv / memory
- 改变
	- storage type 归一化，将服务供应商与协议类型拆分开，
		- aws_s3 => s3
		- azure_blob => azblob
		- local_fs => fs
	- storage 接口重新设计 -> DAL2
		- allow adding new type
			- fs/[[nfs]]/nas/[[s3]]/[[gcs]]/oss/[[ipfs]]
	- format trait -> allow adding new format
		- httpd/nginx log
		- mysql dump files
- 现在的进展
	- DAL2 refactor
		- implement s3 support
- 更远的想法
	- Query Pushdown
		- S3 Selcet
		- Big Query?
	- Cloud Native Storage Format
		- parquet 不是面向 non-seekable storage 设计的
			- 大量的 seek 操作导致请求量增加，维护内部的 buffer 实现复杂，不利于性能优化
		- 我们需要自行设计或者改造一个新的 storage format
- 参考资料
	- [Prestodb Connectors](https://prestodb.io/docs/current/connector.html)
	- [[Apache Flink/Connectors]]
	- [Snowpipe](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro.html)
-