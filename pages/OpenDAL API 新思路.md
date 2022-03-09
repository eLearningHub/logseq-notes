- ```rust
  async fn read(&self, w: BoxedAsyncWrite, args: &OpRead) -> Result<usize>
  ```
-
- 其实可以考虑底层支持多种 API
	- ```rust
	  async fn read(&self, w: BoxedAsyncWrite, args: &OpRead) -> Result<usize>
	  ```
	- ```rust
	  async fn read(&self, sender: tokio::sync::mpsc::Sender<bytes>, args: &OpRead) -> Result<usize>
	  ```
	- 这样会导致 service 实现的复杂度高很多
		- 需要 bench 一下 channel 发送 bytes 的开销到底有多大
-