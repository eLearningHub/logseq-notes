- Trailer 用来在 Chunked Message 后面附加 headers
-
- 语法
	- ```text
	  Trailer: header-names
	  ```
- 示例
	- ```http
	  HTTP/1.1 200 OK
	  Content-Type: text/plain
	  Transfer-Encoding: chunked
	  Trailer: Expires
	  
	  7\r\n
	  Mozilla\r\n
	  9\r\n
	  Developer\r\n
	  7\r\n
	  Network\r\n
	  0\r\n
	  Expires: Wed, 21 Oct 2015 07:28:00 GMT\r\n
	  \r\n
	  ```
- 参考资料
	- [MDN: Trailer](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Trailer)
-