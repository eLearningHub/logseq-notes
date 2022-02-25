title:: 2022-08: Iteration 7 汇报
type:: [[Blog]]

- [[Iteration/7]]
-
- 用一个词语来概括这个周期就是兵荒马乱。
-
- 这周 opendal 正式上线，替代了原本 databend 中的 `common/dal`。随着 opendal 开始进入真实的场景，很多问题被一下子暴露了出来：性能不好，API 难用， 报错信息过于简陋。其中最难堪的莫过于性能问题了：根据反馈，opendal 上线后，databend 的吞吐下降到了旧版本的 1%。好在大家都比较 nice，讨论问题对事不对人，不然遭遇职场滑铁卢的我要羞愧辞职了。(我好菜啊- -)
-
- 背后的根本原因是我对 Async Rust 的理解还是不到位。有些概念自己学过了，理解过了，但是到写时候，在一个真正的复杂环境了，往往会忽略了它与其他因素的交互。在这次的大型性能回退(以及后续的修复)中，我陆陆续续犯了很多错误，这里挑一些记忆比较深刻的。
-
- 先看第一个版本的错误实现：
- ```rust
   match &mut self.state {
              ReadState::Idle => {
                  let acc = self.acc.clone();
                  let pos = self.pos;
                	let size = min(buf.len(), self.remaining);
                  let op = OpRead {
                      path: self.path.to_string(),
                      offset: Some(pos),
                      size: Some(size),
                  };
  
                  let future = async move { acc.read(&op).await };
  
                  self.state = ReadState::Sending(Box::pin(future));
                  self.poll_read(cx, buf)
              }
              ReadState::Sending(future) => match ready!(Pin::new(future).poll(cx)) {
                  Ok(r) => {
                      self.state = ReadState::Reading(r);
                      self.poll_read(cx, buf)
                  }
                  Err(e) => Poll::Ready(Err(io::Error::from(e))),
              },
              ReadState::Reading(r) => match ready!(Pin::new(r).poll_read(cx, buf)) {
                  Ok(n) => {
                      self.pos += n as u64;
                      self.state = ReadState::Idle;
                      Poll::Ready(Ok(n))
                  }
                  Err(e) => Poll::Ready(Err(e)),
              },
          }
  }
  ```
- 逻辑很简单，`Idle` 状态时构造一个 future，`Sending` 时尝试 resolve 它，`Reading` 时则读取它并重置。看起来挺对的，但是这个版本会慢到爆炸，因为每次 IO 只有 8k 不到。底层的 Reader 并不会保证每次都把 `buf` 填满。
- 好，现在定位到性能差的原因是每次 IO 粒度太小，是否能够尝试每次都把 buf 填满再返回呢？
- ```rust
   match &mut self.state {
              ReadState::Idle => {
                  let acc = self.acc.clone();
                  let pos = self.pos;
                	let size = min(buf.len(), self.remaining);
                  let op = OpRead {
                      path: self.path.to_string(),
                      offset: Some(pos),
                      size: Some(size),
                  };
  
                  let future = async move { acc.read(&op).await };
  
                  self.state = ReadState::Sending(Box::pin(future));
                  self.poll_read(cx, buf)
              }
              ReadState::Sending(future) => match ready!(Pin::new(future).poll(cx)) {
                  Ok(r) => {
                      self.state = ReadState::Reading(r);
                      self.poll_read(cx, buf)
                  }
                  Err(e) => Poll::Ready(Err(io::Error::from(e))),
              },
              ReadState::Reading(r) => match ready!(Pin::new(r).read_exact(buf).poll(cx)) {
                  Ok(n) => {
                      self.pos += n as u64;
                      self.state = ReadState::Idle;
                      Poll::Ready(Ok(n))
                  }
                  Err(e) => Poll::Ready(Err(e)),
              },
          }
  }
  ```
- 这就是一个典型错误了，`r.read_exact(buf)` 会构造了一个新的 future，这导致每次 runtime poll 进来的时候都在尝试 resolve 一个新的 future 直至报错。如果尝试自己 loop 这个 future 就更错了：现在 async rust 是协作式调度，我们在实现 future 的时候要尽快的返回 Poll::Pending。如果在函数中执行阻塞操作，就会阻塞住整个 runtime，没有线程进行调度，结果的体现就是 Hang 住了。
- 现在需要跳出来想一想用户是如何使用我们的 Reader ：
	- 构造一个 reader
	- 读取一些数据
	- drop 这个 reader
- 所以我们真正要做的事情是在整个 reader 的生命周期内，只发起一次 `read` 请求，让用户始终在 poll_read 同一个底层的 reader。所以在发起请求的时候，只需要指定正确的 offset，不需要限制 size，把 buffer 之类的任务交给用户来决定。
- ```rust
          match &mut self.state {
              ReadState::Idle => {
                  let acc = self.acc.clone();
                  let pos = self.pos;
                  let op = OpRead {
                      path: self.path.to_string(),
                      offset: Some(pos),
                      size: None,
                  };
  
                  let future = async move { acc.read(&op).await };
  
                  self.state = ReadState::Sending(Box::pin(future));
                  self.poll_read(cx, buf)
              }
              ReadState::Sending(future) => match ready!(Pin::new(future).poll(cx)) {
                  Ok(r) => {
                      self.state = ReadState::Reading(r);
                      self.poll_read(cx, buf)
                  }
                  Err(e) => Poll::Ready(Err(io::Error::from(e))),
              },
              ReadState::Reading(r) => match ready!(Pin::new(r).poll_read(cx, buf)) {
                  Ok(n) => {
                      self.pos += n as u64;
                      Poll::Ready(Ok(n))
                  }
                  Err(e) => Poll::Ready(Err(e)),
              },
          }
      }
  ```
- 用户的每次 `poll_read` 返回 `Poll::Ready` 后，这一次的 future 就已经处理完毕了，但是 `Reader` 并没有被 drop，同一个 Reader 的下一次 `poll_read` 请求还是会进入到 `Reading` 状态。我之所以犯了上面的种种错误，就是因为没有搞清楚在 Future 的状态变化中，哪些是在变化的，而哪些是不变的。
-
- 在这些问题都修复之后，databend 的性能终于回归了预期的水平，可以腾出手做