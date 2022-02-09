- [[AWS]] 的 [[Rust]] SDK 可以说是非常难用了。。
-
- 一个简单的自定义 endpoint 的需求都折腾了半天
-
- ```rust
      /// Sets the endpoint resolver to use when making requests.
      pub fn endpoint_resolver(
          mut self,
          endpoint_resolver: impl aws_endpoint::ResolveAwsEndpoint + 'static,
      ) -> Self {
          self.endpoint_resolver = Some(::std::sync::Arc::new(endpoint_resolver));
          self
      }
  ```
- ```rust
  pub trait ResolveAwsEndpoint: Send + Sync {
      // TODO: consider if we want modeled error variants here
      fn resolve_endpoint(&self, region: &Region) -> Result<AwsEndpoint, BoxError>;
  }
  ```
-
- 理论上我可以
	- ```rust
	  use aws_smithy_http::endpoint::Endpoint;
	  use http::Uri;
	  let config = dynamodb::Config::builder()
	      .endpoint(Endpoint::immutable(Uri::from_static("http://localhost:8080"))
	  );
	  ```
	- 但是这还不够，我需要能够控制
		- region
		- disable_ssl
		- path_style
		- sig_v4/v2
- https://docs.aws.amazon.com/sdk-for-rust/latest/dg/s3-object-lambda.html
	- 但是这个 uri_template 要求 `&'static str`
	- 太奇怪了。。