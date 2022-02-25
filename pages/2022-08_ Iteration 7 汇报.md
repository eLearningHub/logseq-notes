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
-