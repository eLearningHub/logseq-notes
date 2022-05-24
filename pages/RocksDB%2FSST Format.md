title:: RocksDB/SST Format

- [Block-based table](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)
	- ``` text
	  	  <beginning_of_file>
	  	  [data block 1]
	  	  [data block 2]
	  	  ...
	  	  [data block N]
	  	  [meta block 1: filter block]                  (see section: "filter" Meta Block)
	  	  [meta block 2: index block]
	  	  [meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
	  	  [meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
	  	  [meta block 5: stats block]                   (see section: "properties" Meta Block)
	  	  ...
	  	  [meta block K: future extended block]  (we may add more meta blocks in the future)
	  	  [metaindex block]
	  	  [Footer]                               (fixed size; starts at file_size - sizeof(Footer))
	  	  <end_of_file>
	  ```
- [PlainTable Format](https://github.com/facebook/rocksdb/wiki/PlainTable-Format)
	- ``` text
	  	      <beginning_of_file>
	  	        [data row1]
	  	        [data row1]
	  	        [data row1]
	  	        ...
	  	        [data rowN]
	  	        [Property Block]
	  	        [Footer]                               (fixed size; starts at file_size - sizeof(Footer))
	  	      <end_of_file>
	  ```
-