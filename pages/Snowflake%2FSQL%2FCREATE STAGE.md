title:: Snowflake/SQL/CREATE STAGE

- Snowflake 使用 STAGE 来关联一个存储服务的特定信息，其中
	- Internal stage：是 Snowflake 内部用来存储数据的 Stage (永久或临时)
	- External Stage：是用户来提供的外部存储，比如 [[AWS S3]]， [[Azure Storage]] 等
-
- 目前支持的文件类型
	- [[CSV]]
	- [[JSON]]
	- [[Avro]]
	- [[Orc]]
	- [[Parquet]]
	- [[XML]]
- 目前支持的存储服务类型
	- [[AWS S3]]
	- [[Google Cloud Storage]]
	- [[Azure Blobs]]
-
- 样例
	- AWS S3
		- ```sql
		  create stage my_ext_stage1
		    url='s3://load/files/'
		    credentials=(aws_key_id='1a2b3c' aws_secret_key='4x5y6z');
		  ```
	- GCS
		- ```sql
		  create stage mystage
		    url='gcs://load/files/'
		    storage_integration = my_storage_int
		    directory = (
		      enable = true
		      auto_refresh = true
		      notification_integration = 'MY_NOTIFICATION_INT'
		    );
		  ```
	- Azblob
		- ```sql
		  create stage mystage
		    url='azure://myaccount.blob.core.windows.net/mycontainer/files/'
		    credentials=(azure_sas_token='?sv=2016-05-31&ss=b&srt=sco&sp=rwdl&se=2018-06-27T10:05:50Z&st=2017-06-27T02:05:50Z&spr=https,http&sig=bgqQwoXwxzuD2GJfagRg7VOS8hzNr3QLT7rhS8OFRLQ%3D')
		    encryption=(type='AZURE_CSE' master_key = 'kPxX0jzYfIamtnJEUTHwq80Au6NbSgPH5r4BDDwOaO8=')
		    file_format = my_csv_format;
		  ```
		- ```sql
		  create stage mystage
		    url='azure://myaccount.blob.core.windows.net/load/files/'
		    storage_integration = my_storage_int
		    directory = (
		      enable = true
		      auto_refresh = true
		      notification_integration = 'MY_NOTIFICATION_INT'
		    );
		  ```
-
- 疑问
	- create stage 时候指定的 `directory.auto_refresh` 是什么功能？ #question
-
- 参考资料
	- [Create Stage](https://docs.snowflake.com/en/sql-reference/sql/create-stage.html)