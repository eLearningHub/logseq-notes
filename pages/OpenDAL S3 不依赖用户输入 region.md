- 理论上 S3 能够不依赖用户自己输入的 region 来判断
-
- list_buckets => 需要权限
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
- 用户没给 endpoint
	- 发送一个 HEAD 请求给 `<bucket>.<endpoint>`
		- 然后取 `X-Amz-Bucket-Region`
		- 404 说明 bucket 不存在，可能的响应包括 200 和 403
- 用户给了 endpoint
	- 发送 HEAD 请求给 `<bucket>.<endpoint>`
		-