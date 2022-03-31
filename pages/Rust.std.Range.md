title:: Rust/std/Range

- Rust 对 range 的操作提供了原生的支持，根据区间开闭的情况可以划分为
	- `Range` => `start..end`
	- `RangeFrom` => `start..`
	- `RangeFull` => `..`
	- `RangeInclusive` => `start..=end`
	- `RangeTo` => `..end`
	- `RangeToInclusive` => `..=end`
- 这些结构体都实现了 Index，Hash 等 trait，我们可以自由地用于很多场景，比如
	- 切片 => `arr[1..]` / `arr[..n]`
	- 迭代器 => `for i in 1..10` / `(1..10).sum()`
-
- 所有 range 相关的结构体都实现了 [`RangeBounds`](https://doc.rust-lang.org/std/ops/trait.RangeBounds.html)
	- id:: 6245af57-13ba-40f3-8231-8f82f84d03bb
	  ```rust
	  pub enum Bound<T> {
	      Included(T),
	      Excluded(T),
	      Unbounded,
	  }
	  
	  pub trait RangeBounds<T> 
	  where
	      T: ?Sized, 
	  {
	      fn start_bound(&self) -> Bound<&T>;
	      fn end_bound(&self) -> Bound<&T>;
	  
	      fn contains<U>(&self, item: &U) -> bool
	      where
	          T: PartialOrd<U>,
	          U: PartialOrd<T> + ?Sized,
	      { ... }
	  }
	  ```
	- 可以调用 `start_bound`，`end_bound` 来获取这个区间的开闭情况
- 所以我们在设计想要接收一个 range 的函数时，可以这样
	- ```rust
	  pub fn range_mut<T, R>(&mut self, range: R) -> RangeMut<'_, K, V>
	  where
	      T: Ord + ?Sized,
	      K: Borrow<T> + Ord,
	      R: RangeBounds<T>, 
	  ```
	- 要求用户输入一个 `impl RangeBounds<T>`，这样用起来就会很舒服
	- ```rust
	  use std::collections::BTreeMap;
	  
	  let mut map: BTreeMap<&str, i32> = ["Alice", "Bob", "Carol", "Cheryl"]
	      .iter()
	      .map(|&s| (s, 0))
	      .collect();
	  for (_, balance) in map.range_mut("B".."Cheryl") {
	      *balance += 100;
	  }
	  for (name, balance) in &map {
	      println!("{} => {}", name, balance);
	  }
	  ```
-
- 参考资料
	- [Rust std::ops::Bound](https://doc.rust-lang.org/std/ops/enum.Bound.html)
	- [Rust std::ops::Range](https://doc.rust-lang.org/std/ops/struct.Range.html)