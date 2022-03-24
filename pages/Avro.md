- https://avro.apache.org/
-
- Avro 是 [[Hadoop]] 的子项目，现在已经独立出来成为 [[Apache 基金会]] 的独立项目
-
- Avro 是一个基于二进制传输的高性能 RPC和数据序列化系统(data serialization system)
	- 使用JSON定义数据类型及通信协议
	- 使用压缩二进制来序列化数据
	- 是 [[Hadoop]] 持久化数据的一种序列化格式
	- [[HBase]] 和 [[Hive]] 的客户端与服务器端传输也用到了。
-
- 特性
	- 丰富的数据结构类型
	- 快速可压缩的二进制数据形式
	- 存储持久数据的文件容器
	- 远程过程调用 RPC
	- 简单的动态语言结合功能
		- Avro 和动态语言结合后，读写数据文件和使用 RPC 协议都不需要生成代码
		- 代码生成作为一种可选的优化可以只在静态类型语言中实现。
-
- 样例
	- ```avsc
	  {
	   "namespace": "com.datalimbo.qos.avro",
	   "type": "record",
	   "name": "User",
	   "fields": [
	       {"name": "name", "type": "string"},
	       {"name": "favorite_number",  "type": ["int", "null"]},
	       {"name": "favorite_color", "type": ["string", "null"]}
	   ]
	  }
	  ```
-
- Rust 实现
	- [avro_rs](https://docs.rs/avro-rs/latest/avro_rs/)
		- 解析 Schema
			- ```rust
			  use avro_rs::Schema;
			  
			  let raw_schema = r#"
			      {
			          "type": "record",
			          "name": "test",
			          "fields": [
			              {"name": "a", "type": "long", "default": 42},
			              {"name": "b", "type": "string"}
			          ]
			      }
			  "#;
			  
			  // if the schema is not valid, this function will return an error
			  let schema = Schema::parse_str(raw_schema).unwrap();
			  
			  // schemas can be printed for debugging
			  println!("{:?}", schema);
			  ```
		- 写入数据
			- ```rust
			  use avro_rs::types::Record;
			  use avro_rs::Writer;
			  // a writer needs a schema and something to write to
			  let mut writer = Writer::new(&schema, Vec::new());
			  
			  // the Record type models our Record schema
			  let mut record = Record::new(writer.schema()).unwrap();
			  record.put("a", 27i64);
			  record.put("b", "foo");
			  
			  // schema validation happens here
			  writer.append(record).unwrap();
			  
			  // this is how to get back the resulting avro bytecode
			  // this performs a flush operation to make sure data has been written, so it can fail
			  // you can also call `writer.flush()` yourself without consuming the writer
			  let encoded = writer.into_inner().unwrap();
			  ```
		- 同样支持 [[serde]]
			- ```rust
			  use avro_rs::Writer;
			  
			  #[derive(Debug, Serialize)]
			  struct Test {
			      a: i64,
			      b: String,
			  }
			  
			  // a writer needs a schema and something to write to
			  let mut writer = Writer::new(&schema, Vec::new());
			  
			  // the structure models our Record schema
			  let test = Test {
			      a: 27,
			      b: "foo".to_owned(),
			  };
			  
			  // schema validation happens here
			  writer.append_ser(test).unwrap();
			  
			  // this is how to get back the resulting avro bytecode
			  // this performs a flush operation to make sure data is written, so it can fail
			  // you can also call `writer.flush()` yourself without consuming the writer
			  let encoded = writer.into_inner();
			  ```
- 参考资料
	- [Apache avro 简介](https://zhuanlan.zhihu.com/p/53892693)