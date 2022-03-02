- [[AWS]] 旗下的交互式数据分析服务
	- Built on [[Presto]], runs standard SQL
- 产品页 https://aws.amazon.com/athena/
- 文档页 https://docs.aws.amazon.com/athena/latest/ug/getting-started.html
-
- Create Database
	- Table Location in Amazon S3
		- https://docs.aws.amazon.com/athena/latest/ug/tables-location-format.html
		- ```sql
		  CREATE EXTERNAL TABLE `test_table`(
		  ...
		  )
		  ROW FORMAT ...
		  STORED AS INPUTFORMAT ...
		  OUTPUTFORMAT ...
		  LOCATION s3://bucketname/folder/
		  ```
-
- 参考资料
	- [Top 10 Performance Tuning Tips for Amazon Athena](https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/)
	- [Supported SerDes and Data Formats](https://docs.aws.amazon.com/athena/latest/ug/supported-serdes.html)
		- csv
		- tsv
		- json
-