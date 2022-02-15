title:: Rust/Bench

- Rust 自带的 bench 还没有 stable，而且对 Async 的支持不太好，社区使用 [[bheisler/criterion.rs]] 比较多。
-
- 在 `Cargo.toml` 中增加
	- ```toml
	  [[bench]]
	  name = "my_test"
	  harness = false
	  
	  [dev-dependencies]
	  criterion = { version = "0.3", features = ["async", "async_tokio"] }
	  ```
- 在文件 `benches/my_test.rs` 中增加测试：
	- ```rust
	  use criterion::{criterion_group, criterion_main, Criterion};
	  
	  async fn test() {
	      something_async().await
	  }
	  
	  fn criterion_benchmark(c: &mut Criterion) {
	      let runtime = tokio::runtime::Runtime::new().unwrap();
	  
	      c.bench_function("test", |b| {
	          b.to_async(&runtime).iter(|| test());
	      });
	  }
	  
	  criterion_group!(benches, criterion_benchmark);
	  criterion_main!(benches);
	  ```
- 然后运行 `cargo bench` 即可
-
- [[bheisler/criterion.rs]] 会自动记录之前的 bench 结果，然后展示两次 bench 之间的差值，还是挺好用的
	- ![image.png](../assets/image_1644930943206_0.png)
-