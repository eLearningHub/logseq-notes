title:: antonok/enum_dispatch

- https://gitlab.com/antonok/enum_dispatch
- https://docs.rs/enum_dispatch/latest/enum_dispatch/index.html
-
- ```rust
  #[enum_dispatch]
  enum MyBehaviorEnum {
      MyImplementorA,
      MyImplementorB,
  }
  
  #[enum_dispatch(MyBehaviorEnum)]
  trait MyBehavior {
      fn my_trait_method(&self);
  }
  
  let a: MyBehaviorEnum = MyImplementorA::new().into();
  
  a.my_trait_method();    //no dynamic dispatch
  ```
-
- 避免动态分发 trait 的方法，适合不需要外部来实现 trait 的情况
	- [[OpenDAL]] 应该用不了这个，因为需要支持 Layer
-