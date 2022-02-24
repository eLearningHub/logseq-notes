- 已经提交为 RFC: https://github.com/datafuselabs/opendal/pull/57
-
- 理论上 S3 能够不依赖用户自己输入的 region 来判断
-
- list_buckets => 需要权限，不太现实
	- 最好不要 list_buckets
- 可以发送一个 HEAD 请求来看
	- 访问 `s3.amazonaws.com/databend-shared` 会返回
		- HTTP/1.1 200 Connection established
		- ```http
		  < HTTP/1.1 301 Moved Permanently
		  < x-amz-bucket-region: us-east-2
		  < x-amz-request-id: VXEGYW7W3YEEX22Z
		  < x-amz-id-2: 9WMUoRYwNnipgmGZbZIHkRgjz1qWbiTZapwWhAD+ATnYyz7LYzbf1QeWJUpmmFY9vCM4S7kQxmL5akWMkBXxtw==
		  < Content-Type: application/xml
		  < Transfer-Encoding: chunked
		  < Date: Thu, 24 Feb 2022 03:40:10 GMT
		  < Server: AmazonS3
		  <
		  <?xml version="1.0" encoding="UTF-8"?>
		  * Connection #0 to host 127.0.0.1 left intact
		  <Error><Code>PermanentRedirect</Code><Message>The bucket you are attempting to access must be addressed using the specified endpoint. Please send all future requests to this endpoint.</Message><Endpoint>databend-shared.s3.amazonaws.com</Endpoint><Bucket>databend-shared</Bucket><RequestId>VXEGYW7W3YEEX22Z</RequestId><HostId>9WMUoRYwNnipgmGZbZIHkRgjz1qWbiTZapwWhAD+ATnYyz7LYzbf1QeWJUpmmFY9vCM4S7kQxmL5akWMkBXxtw==</HostId></Error>
		  ```
	- 现在访问 `databend-shared.s3.amazonaws.com` 会返回
		- ```http
		  < HTTP/1.1 403 Forbidden
		  < Transfer-Encoding: chunked
		  < Connection: keep-alive
		  < Content-Type: application/xml
		  < Date: Thu, 24 Feb 2022 03:31:40 GMT
		  < Keep-Alive: timeout=4
		  < Proxy-Connection: keep-alive
		  < Server: AmazonS3
		  < X-Amz-Bucket-Region: us-east-2
		  < X-Amz-Id-2: mTHHmaUuBzAexc4WLicigUc63cqVOmVWox3fYOypVuMpTA9oVcSCZKpBYQgKwxTnQ2uNAjpt42E=
		  < X-Amz-Request-Id: 3EWMB6XK7K02RTJ2
		  ```
-
- 判断流程
	- 用户没给 endpoint，按照 `https://s3.amazonaws.com` 处理
	- 发送一个 HEAD 请求给 `<endpoint>/<bucket>`
		- 标准的 S3 服务会提供 `X-Amz-Bucket-Region` 指示正确的 region
	- 服务返回 200 / 403
		- 说明当前 endpoint 本来就是完整的
		- 响应中如果有 `X-Amz-Bucket-Region` 可以直接使用
		- 如果没有就需要 fallback 到 `us-east-1`
	- 服务返回 301
		- 说明 endpoint 缺少了 region，需要使用 `X-Amz-Bucket-Region` 补全
		- 没有的话需要报错
	- 其他的状态码说明 bucket 不存在或者 endpoint 有问题
-
- 兼容情况
	- [[minio]] 提供了这个兼容
		- 在启动 minio 的时候设置 region：
			- ```shell
			  MINIO_SITE_REGION=test minio server . 
			  ```
		- 那么去访问他的时候也会返回 `X-Amz-Bucket-Region`
			- ```shell
			  HTTP/1.1 403 Forbidden
			  Accept-Ranges: bytes
			  Content-Length: 0
			  Content-Security-Policy: block-all-mixed-content
			  Server: MinIO
			  Strict-Transport-Security: max-age=31536000; includeSubDomains
			  Vary: Origin
			  Vary: Accept-Encoding
			  X-Amz-Bucket-Region: test
			  X-Amz-Request-Id: 16D69F6B835C12A8
			  X-Content-Type-Options: nosniff
			  X-Xss-Protection: 1; mode=block
			  Date: Thu, 24 Feb 2022 04:46:37 GMT
			  ```
	- [[Aliyun OSS]]
		- Aliyun 不支持重定向，会直接返回 404
		- 只有 endpoint 正确的，比如 `oss-ap-northeast-1.aliyuncs.com`，才会返回 403 错误
	- [[QingStor]]
		- ```shell
		  curl https://s3.qingstor.com/community
		  
		  HTTP/1.1 301 Moved Permanently
		  Server: nginx/1.13.6
		  Date: Thu, 24 Feb 2022 04:09:41 GMT
		  Connection: keep-alive
		  Location: https://s3.pek3a.qingstor.com/community
		  X-Qs-Request-Id: 05b8281a10004415
		  ```
		- 需要解析 `Location`？