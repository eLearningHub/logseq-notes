- ```rust
  impl futures::AsyncRead for InputStreamInterceptor {
      fn poll_read(
          mut self: Pin<&mut Self>,
          ctx: &mut std::task::Context<'_>,
          buf: &mut [u8],
      ) -> Poll<std::io::Result<usize>> {
          let start = match self.last_pending {
              None => Instant::now(),
              Some(t) => t,
          };
          let r = Pin::new(&mut self.inner).poll_read(ctx, buf);
          match r {
              Poll::Ready(Ok(len)) => {
                  self.ctx.inc_read_bytes(len as usize);
                  self.last_pending = None;
              }
              Poll::Ready(Err(_)) => self.last_pending = None,
              Poll::Pending => self.last_pending = Some(start),
          }
          self.ctx
              .inc_read_byte_cost_ms(start.elapsed().as_millis() as usize);
          r
      }
  }
  ```