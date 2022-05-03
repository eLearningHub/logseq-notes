- 标准库提供了这样的一个例子
	- ```rust
	  fn call_on_ref_zero<F>(f: F) where F: Fn(&i32) {
	      let zero = 0;
	      f(&zero);
	  }
	  ```
	-
- 现在最新的 Stable 已经能正常编译这样的代码了
	- 所以我们需要强化一下它，比如改造成接受两个 ref 并返回其中一个
	- ```rust
	  fn call_on_ref_zero<F>(f: F)
	  where
	      F: Fn(&i32, &i32) -> &i32,
	  {
	      let zero = 0;
	      f(&zero, &zero);
	  }
	  
	  fn main() {
	      call_on_ref_zero(|x, y|->&i32 {
	          println!("{}, {}", x, y);
	  
	          &0
	      })
	  }
	  
	  ```
- 这段代码编译会报错
	- ```rust
	  error[E0106]: missing lifetime specifier
	   --> src/main.rs:3:26
	    |
	  3 |     F: Fn(&i32, &i32) -> &i32,
	    |           ----  ----     ^ expected named lifetime parameter
	    |
	    = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from argument 1 or argument 2
	    = note: for more information on higher-ranked polymorphism, visit https://doc.rust-lang.org/nomicon/hrtb.html
	  help: consider making the bound lifetime-generic with a new `'a` lifetime
	    |
	  3 |     F: for<'a> Fn(&'a i32, &'a i32) -> &'a i32,
	    |        +++++++     ++       ++          ++
	  help: consider introducing a named lifetime parameter
	    |
	  1 ~ fn call_on_ref_zero<'a, F>(f: F)
	  2 | where
	  3 ~     F: Fn(&'a i32, &'a i32) -> &'a i32,
	    |
	  
	  For more information about this error, try `rustc --explain E0106`.
	  error: could not compile `playground` due to previous error
	  ```
- 因为 Rust 此时已经无法推断出这三个引用之间的关系，需要我们显式的提供一些标注
- 运用 HRTB，我们可以修改如下
	- ```rust
	  fn call_on_ref_zero<F>(f: F)
	  where
	      F: for<'a> Fn(&'a i32, &'a i32) -> &'a i32,
	  {
	      let zero = 0;
	      f(&zero, &zero);
	  }
	  
	  fn main() {
	      call_on_ref_zero(|x, y|->&i32 {
	          println!("{}, {}", x, y);
	  
	          &0
	      })
	  }
	  
	  ```
	- 运用 `for<'a>` 这样的语法表示对任意生命周期都成立
- 我们也可以显式的告诉编译器返回的这个值借用自哪个参数
	- ```rust
	  fn call_on_ref_zero<F>(f: F)
	  where
	      F: for<'a, 'b> Fn(&'a i32, &'b i32) -> &'a i32,
	  {
	      let zero = 0;
	      f(&zero, &zero);
	  }
	  
	  fn main() {
	      call_on_ref_zero(|x, y|->&i32 {
	          println!("{}, {}", x, y);
	  
	          &0
	      })
	  }
	  
	  ```
- 参考资料
	- RFC: [0387-higher-ranked-trait-bounds](https://github.com/rust-lang/rfcs/blob/master/text/0387-higher-ranked-trait-bounds.md)
	- Rust References: [Higher-ranked trait bounds](https://doc.rust-lang.org/reference/trait-bounds.html#higher-ranked-trait-bounds)
	- [Higher-Rank Trait Bounds (HRTBs)](https://doc.rust-lang.org/nomicon/hrtb.html)
	- [谈一谈rust里的一个黑魔法语法HRTBs](https://dengjianping.github.io/2019/07/09/%E8%B0%88%E4%B8%80%E8%B0%88rust%E9%87%8C%E7%9A%84%E4%B8%80%E4%B8%AA%E9%BB%91%E9%AD%94%E6%B3%95%E8%AF%AD%E6%B3%95HRTBs.html)
	- [数据库表达式执行的黑魔法：用 Rust 做类型体操 (Part 2)](https://zhuanlan.zhihu.com/p/461405621)