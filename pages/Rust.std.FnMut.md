- 作为函数的范型参数
	- ```rust
	  fn do_twice<F>(mut func: F)
	      where F: FnMut()
	  {
	      func();
	      func();
	  }
	  
	  let mut x: usize = 1;
	  {
	      let add_two_to_x = || x += 2;
	      do_twice(add_two_to_x);
	  }
	  
	  assert_eq!(x, 5);
	  ```
- 或者作为结构体的范型参数
	- ```rust
	  #[pin_project]
	  pub struct CallbackReader<F: FnMut(usize)> {
	      #[pin]
	      inner: Reader,
	      f: F,
	  }
	  
	  let reader = CallbackReader {
	    inner: Box::new(Cursor::new("Hello, world!")),
	    f: |n| size += n,
	  };
	  ```
-