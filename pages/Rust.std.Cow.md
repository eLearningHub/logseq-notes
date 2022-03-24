- https://doc.rust-lang.org/std/borrow/enum.Cow.html
- A clone-on-write smart pointer.
-
- 实现了 `impl<'a> From<String> for Cow<'a, str>`
- ```rust
  let s = "eggplant".to_string();
  let s2 = "eggplant".to_string();
  assert_eq!(Cow::from(s), Cow::<'static, str>::Owned(s2));
  ```