- reqsign 旨在支持在不依赖 SDK 的情况下对请求进行签名
	- 比如说不依赖 AWS S3 SDK 的情况下，可以快速对一个请求进行签名，适合只需要使用个别 API 的场景
-
- 设计目标
	- 正确(确保签名的正确性)
	- 轻量(不引入过多依赖)
	- 零开销(避免复制用户的 request)
-
- v0.1 特性
	- 支持 aws s3 签名(支持加载默认的 credential？)
-
- 设计考虑
	- 怎么支持各种 Request 结构体呢？
	- 不同服务的考虑
		- aws：一套 signing logic
			- https://docs.aws.amazon.com/general/latest/gr/sigv4-create-canonical-request.html
		- gcs：Oauth2
		- azblob：一套 signing logic (类似 aws)
			- https://docs.microsoft.com/en-us/rest/api/storageservices/authorize-with-shared-key
-
- 啊，第一个请求测试成功了！
	- ```rust
	  running 1 test
	  [2022-03-13T16:52:14Z DEBUG reqsign::services::aws::v4] creq: 0c65a2cd6a1a1a4c14d6e96f5c458fc43255ef0f429e8270d768530adcba0030
	  [2022-03-13T16:52:14Z DEBUG reqsign::services::aws::v4] string to sign: AWS4-HMAC-SHA256
	      20220101T120000Z
	      20220101/test/s3/aws4_request
	      0c65a2cd6a1a1a4c14d6e96f5c458fc43255ef0f429e8270d768530adcba0030
	  test services::aws::tests::test_get_object ... ok
	  
	  
	  ```
	- 可以稍微整理一下代码实现了，后面再加上更多的请求测试
-
- SignableRequest -> CanonicalRequest -> SigningContext
-