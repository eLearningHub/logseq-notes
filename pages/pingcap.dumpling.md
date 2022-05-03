- 导出 csv
	- ```
	  bin/dumpling -u root -p my-secret-pw -P 3306 -h 127.0.0.1 -o /tmp/test1 -r 200000 --filetype csv --sql 'select * from test.test'
	  ```
-