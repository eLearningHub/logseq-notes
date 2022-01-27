- [[MySQL]] 支持[[分区表]]功能
- 文档
	- https://dev.mysql.com/doc/refman/8.0/en/partitioning.html
- 在创建的时候指定 partition key
	- ```mysql
	  CREATE TABLE trb3 (id INT, name VARCHAR(50), purchased DATE)
	      PARTITION BY RANGE( YEAR(purchased) ) (
	          PARTITION p0 VALUES LESS THAN (1990),
	          PARTITION p1 VALUES LESS THAN (1995),
	          PARTITION p2 VALUES LESS THAN (2000),
	          PARTITION p3 VALUES LESS THAN (2005)
	      );
	  ```
- 可以使用 ALTER 来修改 partition key
	- ```mysql
	  ALTER TABLE trb3 PARTITION BY KEY(id) PARTITIONS 2;
	  ```
	- 从社区分享来看，是可以对一个普通的表进行 `ALTER ADD PARTITION`，但是文档里面没有找到明确的说明
		- 从本质上来看这个操作应该就是把表复制一遍，中间需要锁表