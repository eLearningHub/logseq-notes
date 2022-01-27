alias:: GAT

- Generic Associated Type，常被缩写为 GAT
-
- [[Rust]] 现有的 [Associated Type]([[Rust/Associated Type]]) 中不允许出现泛型参数
- 比如说这样的代码是无法编译通过的
	- ```rust
	  trait PointerFamily {
	      type PointerType<T>;
	  }
	  ```
- 同理，带有泛型生命周期的关联类型也是无法通过编译的
	- ```rust
	  trait StreamingIterator {
	      type Item<'a>;
	      fn next<'a>(&'a mut self) -> Option<Self::Item<'a>>;
	  }
	  ```
-
- GAT 这个特性就是为了解决这个问题
-
- 参考资料
	- [The push for GATs stabilization](https://blog.rust-lang.org/2021/08/03/GATs-stabilization-push.html)
	- [了解一点关于泛型关联类型(GAT)的事](https://rustmagazine.github.io/rust_magazine_2021/chapter_5/rust-gat.html)