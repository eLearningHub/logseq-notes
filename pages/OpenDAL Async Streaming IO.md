- [[2022-03-28]]
- read 的 benchmark
	- 使用未初始化的 vec ( read 使用 read_to_end)
	- read
		- ```rust
		  read_full/4.00 KiB      time:   [465.64 us 475.43 us 485.54 us]
		                          thrpt:  [8.0451 MiB/s 8.2163 MiB/s 8.3889 MiB/s]
		                   change:
		                          time:   [-1.3980% +1.8916% +5.1699%] (p = 0.25 > 0.05)
		                          thrpt:  [-4.9157% -1.8565% +1.4178%]
		                          No change in performance detected.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  read_full/256 KiB       time:   [565.02 us 578.71 us 592.98 us]
		                          thrpt:  [421.60 MiB/s 432.00 MiB/s 442.46 MiB/s]
		                   change:
		                          time:   [+4.7048% +8.2212% +11.752%] (p = 0.00 < 0.05)
		                          thrpt:  [-10.516% -7.5967% -4.4934%]
		                          Performance has regressed.
		  read_full/4.00 MiB      time:   [3.0481 ms 3.0724 ms 3.0994 ms]
		                          thrpt:  [1.2603 GiB/s 1.2714 GiB/s 1.2815 GiB/s]
		                   change:
		                          time:   [+8.1354% +10.043% +12.063%] (p = 0.00 < 0.05)
		                          thrpt:  [-10.765% -9.1268% -7.5234%]
		                          Performance has regressed.
		  Found 7 outliers among 100 measurements (7.00%)
		    4 (4.00%) high mild
		    3 (3.00%) high severe
		  read_full/16.0 MiB      time:   [31.636 ms 31.742 ms 31.854 ms]
		                          thrpt:  [502.29 MiB/s 504.07 MiB/s 505.75 MiB/s]
		                   change:
		                          time:   [+177.47% +180.47% +183.35%] (p = 0.00 < 0.05)
		                          thrpt:  [-64.708% -64.346% -63.960%]
		  ```
	- 使用已初始化的 vec (read 使用 read_exact)
		- ```rust
		  read_full/4.00 KiB      time:   [459.96 us 473.13 us 485.80 us]
		                          thrpt:  [8.0409 MiB/s 8.2562 MiB/s 8.4927 MiB/s]
		                   change:
		                          time:   [-5.5399% -2.8300% +0.1571%] (p = 0.07 > 0.05)
		                          thrpt:  [-0.1569% +2.9124% +5.8648%]
		                          No change in performance detected.
		  read_full/256 KiB       time:   [571.27 us 585.38 us 600.31 us]
		                          thrpt:  [416.45 MiB/s 427.07 MiB/s 437.62 MiB/s]
		                   change:
		                          time:   [-1.9701% +1.1744% +4.7728%] (p = 0.48 > 0.05)
		                          thrpt:  [-4.5553% -1.1608% +2.0097%]
		                          No change in performance detected.
		  read_full/4.00 MiB      time:   [2.7202 ms 2.7561 ms 2.7925 ms]
		                          thrpt:  [1.3988 GiB/s 1.4173 GiB/s 1.4360 GiB/s]
		                   change:
		                          time:   [-11.700% -10.294% -8.8648%] (p = 0.00 < 0.05)
		                          thrpt:  [+9.7271% +11.475% +13.250%]
		                          Performance has improved.
		  Found 5 outliers among 100 measurements (5.00%)
		    3 (3.00%) low mild
		    2 (2.00%) high mild
		  read_full/16.0 MiB      time:   [10.298 ms 10.464 ms 10.631 ms]
		                          thrpt:  [1.4698 GiB/s 1.4933 GiB/s 1.5173 GiB/s]
		                   change:
		                          time:   [-67.564% -67.035% -66.428%] (p = 0.00 < 0.05)
		                          thrpt:  [+197.87% +203.35% +208.30%]
		                          Performance has improved.
		  
		  ```
	- read2 (使用已初始化的 vec，copy_from_slice)
		- ```rust
		  read_full/4.00 KiB      time:   [474.43 us 487.28 us 500.02 us]
		                          thrpt:  [7.8122 MiB/s 8.0164 MiB/s 8.2336 MiB/s]
		                   change:
		                          time:   [-1.3426% +1.8369% +5.0575%] (p = 0.27 > 0.05)
		                          thrpt:  [-4.8141% -1.8037% +1.3609%]
		                          No change in performance detected.
		  read_full/256 KiB       time:   [584.21 us 598.16 us 611.34 us]
		                          thrpt:  [408.94 MiB/s 417.95 MiB/s 427.93 MiB/s]
		                   change:
		                          time:   [-1.6743% +1.4823% +4.8773%] (p = 0.37 > 0.05)
		                          thrpt:  [-4.6505% -1.4606% +1.7028%]
		                          No change in performance detected.
		  read_full/4.00 MiB      time:   [2.7750 ms 2.8063 ms 2.8395 ms]
		                          thrpt:  [1.3757 GiB/s 1.3920 GiB/s 1.4077 GiB/s]
		                   change:
		                          time:   [-9.8935% -8.6609% -7.3303%] (p = 0.00 < 0.05)
		                          thrpt:  [+7.9102% +9.4821% +10.980%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  read_full/16.0 MiB      time:   [11.071 ms 11.131 ms 11.195 ms]
		                          thrpt:  [1.3957 GiB/s 1.4038 GiB/s 1.4113 GiB/s]
		                   change:
		                          time:   [-65.156% -64.933% -64.706%] (p = 0.00 < 0.05)
		                          thrpt:  [+183.33% +185.17% +186.99%]
		                          Performance has improved.
		  
		  ```
	- read2 (使用未初始化的 vec, buf.write)
		- ```rust
		  read_full/4.00 KiB      time:   [480.69 us 492.27 us 503.97 us]
		                          thrpt:  [7.7510 MiB/s 7.9352 MiB/s 8.1264 MiB/s]
		                   change:
		                          time:   [-1.1472% +2.0375% +5.2204%] (p = 0.20 > 0.05)
		                          thrpt:  [-4.9614% -1.9968% +1.1605%]
		                          No change in performance detected.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  read_full/256 KiB       time:   [523.07 us 533.92 us 545.30 us]
		                          thrpt:  [458.46 MiB/s 468.24 MiB/s 477.95 MiB/s]
		                   change:
		                          time:   [-8.3360% -5.3778% -2.3290%] (p = 0.00 < 0.05)
		                          thrpt:  [+2.3845% +5.6835% +9.0941%]
		                          Performance has improved.
		  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 8.5s, enable flat sampling, or reduce sample count to 50.
		  read_full/4.00 MiB      time:   [1.6405 ms 1.6811 ms 1.7214 ms]
		                          thrpt:  [2.2692 GiB/s 2.3236 GiB/s 2.3811 GiB/s]
		                   change:
		                          time:   [-44.944% -43.535% -42.122%] (p = 0.00 < 0.05)
		                          thrpt:  [+72.777% +77.099% +81.632%]
		                          Performance has improved.
		  Found 4 outliers among 100 measurements (4.00%)
		    1 (1.00%) low mild
		    3 (3.00%) high mild
		  read_full/16.0 MiB      time:   [6.2463 ms 6.3312 ms 6.4103 ms]
		                          thrpt:  [2.4375 GiB/s 2.4679 GiB/s 2.5015 GiB/s]
		                   change:
		                          time:   [-80.333% -80.054% -79.802%] (p = 0.00 < 0.05)
		                          thrpt:  [+395.10% +401.35% +408.46%]
		                          Performance has improved.
		  ```
- 感觉受 write 实现影响很大，改成用 io::sink() bench 一下看看
- read
	- ```rust
	  read_full/4.00 KiB      time:   [455.70 us 466.18 us 476.93 us]
	                          thrpt:  [8.1904 MiB/s 8.3794 MiB/s 8.5719 MiB/s]
	                   change:
	                          time:   [-5.5291% -2.7958% -0.0514%] (p = 0.05 < 0.05)
	                          thrpt:  [+0.0514% +2.8762% +5.8528%]
	                          Change within noise threshold.
	  read_full/256 KiB       time:   [530.63 us 544.30 us 557.84 us]
	                          thrpt:  [448.16 MiB/s 459.30 MiB/s 471.14 MiB/s]
	                   change:
	                          time:   [-1.5347% +1.1644% +3.9109%] (p = 0.43 > 0.05)
	                          thrpt:  [-3.7637% -1.1510% +1.5586%]
	                          No change in performance detected.
	  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
	  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 9.0s, enable flat sampling, or reduce sample count to 50.
	  read_full/4.00 MiB      time:   [1.5569 ms 1.6152 ms 1.6743 ms]
	                          thrpt:  [2.3330 GiB/s 2.4184 GiB/s 2.5090 GiB/s]
	                   change:
	                          time:   [+2.6452% +6.6427% +10.861%] (p = 0.00 < 0.05)
	                          thrpt:  [-9.7969% -6.2290% -2.5770%]
	                          Performance has regressed.
	  Found 2 outliers among 100 measurements (2.00%)
	    2 (2.00%) high mild
	  read_full/16.0 MiB      time:   [5.7337 ms 5.9087 ms 6.0813 ms]
	                          thrpt:  [2.5693 GiB/s 2.6444 GiB/s 2.7251 GiB/s]
	                   change:
	                          time:   [+6.3767% +10.298% +14.725%] (p = 0.00 < 0.05)
	                          thrpt:  [-12.835% -9.3362% -5.9944%]
	  ```
- read2
- ```rust
  read_full/4.00 KiB      time:   [455.67 us 466.03 us 476.21 us]
                          thrpt:  [8.2027 MiB/s 8.3819 MiB/s 8.5725 MiB/s]
                   change:
                          time:   [-2.1168% +0.6241% +3.8735%] (p = 0.68 > 0.05)
                          thrpt:  [-3.7291% -0.6203% +2.1625%]
                          No change in performance detected.
  Found 1 outliers among 100 measurements (1.00%)
    1 (1.00%) low mild
  read_full/256 KiB       time:   [521.04 us 535.20 us 548.74 us]
                          thrpt:  [455.59 MiB/s 467.11 MiB/s 479.81 MiB/s]
                   change:
                          time:   [-7.8470% -4.7987% -1.4955%] (p = 0.01 < 0.05)
                          thrpt:  [+1.5182% +5.0406% +8.5152%]
                          Performance has improved.
  Found 2 outliers among 100 measurements (2.00%)
    2 (2.00%) high mild
  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 8.8s, enable flat sampling, or reduce sample count to 50.
  read_full/4.00 MiB      time:   [1.4571 ms 1.5184 ms 1.5843 ms]
                          thrpt:  [2.4655 GiB/s 2.5725 GiB/s 2.6808 GiB/s]
                   change:
                          time:   [-5.4403% -1.5696% +2.3719%] (p = 0.44 > 0.05)
                          thrpt:  [-2.3170% +1.5946% +5.7533%]
                          No change in performance detected.
  read_full/16.0 MiB      time:   [5.0201 ms 5.2105 ms 5.3986 ms]
                          thrpt:  [2.8943 GiB/s 2.9988 GiB/s 3.1125 GiB/s]
                   change:
                          time:   [-15.917% -11.816% -7.5219%] (p = 0.00 < 0.05)
                          thrpt:  [+8.1337% +13.400% +18.930%]
                          Performance has improved.
  
  ```
- read2 without async
	- ```rust
	  read_full/4.00 KiB      time:   [470.93 us 486.07 us 501.79 us]
	                          thrpt:  [7.7847 MiB/s 8.0365 MiB/s 8.2947 MiB/s]
	                   change:
	                          time:   [-3.4772% -0.5234% +2.3754%] (p = 0.72 > 0.05)
	                          thrpt:  [-2.3203% +0.5262% +3.6024%]
	                          No change in performance detected.
	  read_full/256 KiB       time:   [537.04 us 556.03 us 574.33 us]
	                          thrpt:  [435.29 MiB/s 449.61 MiB/s 465.52 MiB/s]
	                   change:
	                          time:   [-7.6091% -4.5823% -1.3438%] (p = 0.00 < 0.05)
	                          thrpt:  [+1.3621% +4.8024% +8.2357%]
	                          Performance has improved.
	  Found 1 outliers among 100 measurements (1.00%)
	    1 (1.00%) high mild
	  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
	  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 9.4s, enable flat sampling, or reduce sample count to 50.
	  read_full/4.00 MiB      time:   [1.6280 ms 1.6738 ms 1.7196 ms]
	                          thrpt:  [2.2715 GiB/s 2.3337 GiB/s 2.3995 GiB/s]
	                   change:
	                          time:   [+1.3958% +4.8324% +8.7282%] (p = 0.01 < 0.05)
	                          thrpt:  [-8.0276% -4.6096% -1.3766%]
	                          Performance has regressed.
	  read_full/16.0 MiB      time:   [5.1172 ms 5.2924 ms 5.4677 ms]
	                          thrpt:  [2.8577 GiB/s 2.9523 GiB/s 3.0534 GiB/s]
	                   change:
	                          time:   [-18.535% -15.889% -13.155%] (p = 0.00 < 0.05)
	                          thrpt:  [+15.148% +18.890% +22.751%]
	                          Performance has improved.
	  ```
-
- write 的实现
	- ```rust
	  use std::future::Future;
	  use std::ops::{Deref, DerefMut};
	  use std::pin::Pin;
	  use std::task::{Context, Poll};
	  
	  use anyhow::anyhow;
	  use bytes::Bytes;
	  use futures::channel::mpsc::{Receiver, Sender};
	  use futures::{pending, ready, Sink, StreamExt};
	  use http::{Response, StatusCode};
	  use hyper::client::ResponseFuture;
	  use hyper::Body;
	  
	  use super::error::{Error, Result};
	  
	  pub fn new_channel_body() -> (Sender<Bytes>, hyper::Body) {
	      let (tx, rx) = futures::channel::mpsc::channel(0);
	  
	      (tx, hyper::Body::wrap_stream(rx.map(|v| Ok::<_, Error>(v))))
	  }
	  
	  pub struct BodySinker {
	      path: String,
	      tx: Sender<bytes::Bytes>,
	      fut: ResponseFuture,
	      parse_response: fn(&str, Response<Body>) -> Result<()>,
	  }
	  
	  impl BodySinker {
	      pub fn new(
	          path: &str,
	          tx: Sender<bytes::Bytes>,
	          fut: ResponseFuture,
	          parse_response: fn(&str, Response<Body>) -> Result<()>,
	      ) -> Self {
	          BodySinker {
	              path: path.to_string(),
	              tx,
	              fut,
	              parse_response,
	          }
	      }
	  
	      fn poll_response(
	          mut self: Pin<&mut Self>,
	          cx: &mut Context<'_>,
	      ) -> Poll<std::result::Result<(), Error>> {
	          match Pin::new(&mut self.fut).poll(cx) {
	              Poll::Ready(Ok(resp)) => Poll::Ready((self.parse_response)(&self.path, resp)),
	              // TODO: we need better error output.
	              Poll::Ready(Err(e)) => Poll::Ready(Err(Error::Unexpected(anyhow!(e)))),
	              Poll::Pending => Poll::Pending,
	          }
	      }
	  }
	  
	  impl Sink<bytes::Bytes> for BodySinker {
	      type Error = Error;
	  
	      fn poll_ready(
	          mut self: Pin<&mut Self>,
	          cx: &mut Context<'_>,
	      ) -> Poll<std::result::Result<(), Self::Error>> {
	          let this = &mut *self;
	  
	          if let Poll::Ready(v) = Pin::new(this).poll_response(cx) {
	              unreachable!("response returned too early: {:?}", v)
	          }
	  
	          self.tx
	              .poll_ready(cx)
	              .map_err(|e| Error::Unexpected(anyhow!(e)))
	      }
	  
	      fn start_send(mut self: Pin<&mut Self>, item: Bytes) -> std::result::Result<(), Self::Error> {
	          Pin::new(&mut self.tx)
	              .start_send(item)
	              .map_err(|e| Error::Unexpected(anyhow!(e)))
	      }
	  
	      fn poll_flush(
	          mut self: Pin<&mut Self>,
	          cx: &mut Context<'_>,
	      ) -> Poll<std::result::Result<(), Self::Error>> {
	          Poll::Ready(Ok(()))
	      }
	  
	      fn poll_close(
	          mut self: Pin<&mut Self>,
	          cx: &mut Context<'_>,
	      ) -> Poll<std::result::Result<(), Self::Error>> {
	          if let Err(e) = ready!(Pin::new(&mut self.tx).poll_close(cx)) {
	              return Poll::Ready(Err(Error::Unexpected(anyhow!(e))));
	          }
	  
	          self.poll_response(cx)
	      }
	  }
	  ```