- Sync 用来标记一个类型可以在[[线程]]间共享(通过引用)
	- 跟 [Send]([[Rust/std/Send]]) 一样也是一个 marker trait，
	- `&T: Send` <=> `T: Sync`
-
- 常见的 `!Sync` 的案例
	- 裸指针
		- 这个比较显然
	- [UnsafeCell]([[Rust/std/UnsafeCell]]) 支持 Send 但是被标记了 !Sync
		- 同理，[Cell]([[Rust/std/Cell]]) 和 [RefCell]([[Rust/std/RefCell]]) 也不是 Sync
	- [Rc]([[Rust/std/Rc]])
-
- {{embed ((61f20ff7-ce5e-462f-96be-ee18b449f2ca))}}