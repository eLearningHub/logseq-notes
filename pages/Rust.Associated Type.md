- [[Rust]] 允许我们在 Trait 中指定一个关联类型充当占位符，允许用户在实现 Trait 的时候指定这个具体的类型。
- ```rust
  pub trait Iterator {
      type Item;
  
      fn next(&mut self) -> Option<Self::Item>;
  }
  ```
- 以 `Iterator` 为例
	- ```rust
	  impl Iterator for Counter {
	      type Item = u32;
	  
	      fn next(&mut self) -> Option<Self::Item> { ... }
	  }
	  ```
	- 实现者可以指定 Item 的返回值为 u32。
-
- 跟使用范型的区别在于，范型需要在实现的时候明确指定类型来消除歧义，而关联类型不需要。
	- 假设给定这样的一个 trait
		- ```rust
		  pub trait Iterator<T> {
		      fn next(&mut self) -> Option<T>;
		  }
		  ```
	- 我们可以为同一个类型实现这个 trait 多次
		- `impl Iterator<u64> for Counter`
		- `impl Iterator<i64> for Counter`
	- 但如果使用关联类型，则只能实现一次，在实现的时候就需要明确指定 `Item` 的类型
		- `impl Iterator for Counter`
-
- 参考资料
	- [Specifying Placeholder Types in Trait Definitions with Associated Types](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)
	- Rust Examples: [Associated types](https://doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html)
	- [Generic Parameters v.s. Associated Types](https://zhuanlan.zhihu.com/p/113067733)
-