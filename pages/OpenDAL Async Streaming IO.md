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
-
- ---
- [[2022-03-29]]
- 同时支持返回 Stream 和 Reader 如何？
	- 如何保持面向服务实现者的零开销抽象？
	- 还是本地文件系统就稍微妥协一点？
	- 毕竟大部分服务还是要走网络的
- write 的 benchmark
	- write
		- ```rust
		  rite_once/4.00 KiB     time:   [529.22 us 536.83 us 544.12 us]
		                          thrpt:  [7.1790 MiB/s 7.2765 MiB/s 7.3811 MiB/s]
		                   change:
		                          time:   [-9.3810% -7.0569% -4.6339%] (p = 0.00 < 0.05)
		                          thrpt:  [+4.8591% +7.5927% +10.352%]
		                          Performance has improved.
		  Benchmarking write_once/256 KiB: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 6.4s, enable flat sampling, or reduce sample count to 60.
		  write_once/256 KiB      time:   [1.1969 ms 1.2290 ms 1.2612 ms]
		                          thrpt:  [198.22 MiB/s 203.42 MiB/s 208.88 MiB/s]
		                   change:
		                          time:   [-12.702% -10.491% -8.1839%] (p = 0.00 < 0.05)
		                          thrpt:  [+8.9134% +11.720% +14.551%]
		                          Performance has improved.
		  write_once/4.00 MiB     time:   [11.281 ms 11.390 ms 11.504 ms]
		                          thrpt:  [347.72 MiB/s 351.19 MiB/s 354.58 MiB/s]
		                   change:
		                          time:   [-3.1198% -1.4245% +0.2954%] (p = 0.10 > 0.05)
		                          thrpt:  [-0.2945% +1.4451% +3.2203%]
		                          No change in performance detected.
		  Found 2 outliers among 100 measurements (2.00%)
		    2 (2.00%) high mild
		  write_once/16.0 MiB     time:   [42.197 ms 42.873 ms 43.518 ms]
		                          thrpt:  [367.66 MiB/s 373.19 MiB/s 379.17 MiB/s]
		                   change:
		                          time:   [-1.9646% +0.5363% +3.0966%] (p = 0.66 > 0.05)
		                          thrpt:  [-3.0036% -0.5334% +2.0039%]
		                          No change in performance detected.
		  Found 4 outliers among 100 measurements (4.00%)
		    4 (4.00%) low mild
		  
		  ```
	- write2
		- ```rust
		  write_once/4.00 KiB     time:   [541.23 us 549.21 us 556.74 us]
		                          thrpt:  [7.0162 MiB/s 7.1125 MiB/s 7.2173 MiB/s]
		                   change:
		                          time:   [-0.1558% +1.9535% +4.0942%] (p = 0.07 > 0.05)
		                          thrpt:  [-3.9332% -1.9160% +0.1561%]
		                          No change in performance detected.
		  Found 2 outliers among 100 measurements (2.00%)
		    1 (1.00%) low mild
		    1 (1.00%) high mild
		  Benchmarking write_once/256 KiB: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 6.2s, enable flat sampling, or reduce sample count to 60.
		  write_once/256 KiB      time:   [1.2194 ms 1.2365 ms 1.2545 ms]
		                          thrpt:  [199.28 MiB/s 202.18 MiB/s 205.02 MiB/s]
		                   change:
		                          time:   [-2.3582% +0.0130% +2.5438%] (p = 1.00 > 0.05)
		                          thrpt:  [-2.4807% -0.0130% +2.4152%]
		                          No change in performance detected.
		  write_once/4.00 MiB     time:   [10.218 ms 10.367 ms 10.521 ms]
		                          thrpt:  [380.18 MiB/s 385.86 MiB/s 391.47 MiB/s]
		                   change:
		                          time:   [-10.596% -8.9852% -7.3440%] (p = 0.00 < 0.05)
		                          thrpt:  [+7.9261% +9.8722% +11.851%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  write_once/16.0 MiB     time:   [42.636 ms 43.546 ms 44.430 ms]
		                          thrpt:  [360.12 MiB/s 367.42 MiB/s 375.27 MiB/s]
		                   change:
		                          time:   [-1.2533% +1.5698% +4.1321%] (p = 0.24 > 0.05)
		                          thrpt:  [-3.9681% -1.5455% +1.2692%]
		                          No change in performance detected.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) low mild
		  
		  
		  ```
- 突然想到，要是 read 返回一个 reader，但是 write 返回一个 sink 呢？
	- 是不是也有意思的？
	- Writer？
- ---
- [[2022-03-30]]
-
- Stream/Sink 和 Reader/Writer 到底谁更底层一些？
-
- Stream => Reader，零开销
- Reader => Stream，需要内部的 buffer
- Sink => Writer，需要内部的 buffer？
	- 需要吗？是不是每次 poll_write 都发送一个 send 就行？
		- 虽然不需要额外的 buffering，但是每次 send 的时候需要 clone 一遍输入的数据
- Writer => Sink，零开销
-
- 所以
	- 如果 Read 返回一个 Stream，可以零开销的转换成 Reader
	- 如果 Write 返回一个 Writer，同样可以零开销的转换成 Sink
- 反过来的话都需要额外的 buffering
-
- 再考虑一下内部实现
	- s3 write 能返回一个 writer 嘛？
	- 感觉不太行，这样就需要额外的再复制一遍
-
- 这么看返回 Stream/Sink 已经是比较好的做法了
-
- 要不要把 BytesStream 包装成结构体？
	- 在里面内嵌一个 Stream？并在这个基础上实现 AsyncRead？
	- 感觉没有什么帮助
-
- 实现一组 IntoStream/IntoSink/IntoRead/IntoWrite 会好一些吗？
-
- 如果全都改成输入呢？是不是 wrap 不太好做？
	- 想象一下解压缩的场景
		- 没啥问题
-
- 考虑一下性能
	- read
		- ```rust
		  read_full/4.00 KiB      time:   [488.29 us 497.35 us 506.04 us]
		                          thrpt:  [7.7192 MiB/s 7.8541 MiB/s 7.9998 MiB/s]
		                   change:
		                          time:   [-1.7935% +0.8526% +3.5562%] (p = 0.54 > 0.05)
		                          thrpt:  [-3.4341% -0.8454% +1.8262%]
		                          No change in performance detected.
		  read_full/256 KiB       time:   [588.54 us 599.71 us 610.25 us]
		                          thrpt:  [409.67 MiB/s 416.87 MiB/s 424.78 MiB/s]
		                   change:
		                          time:   [+1.9739% +5.2233% +8.6033%] (p = 0.00 < 0.05)
		                          thrpt:  [-7.9218% -4.9641% -1.9357%]
		                          Performance has regressed.
		  Found 4 outliers among 100 measurements (4.00%)
		    4 (4.00%) low mild
		  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 8.5s, enable flat sampling, or reduce sample count to 50.
		  read_full/4.00 MiB      time:   [1.6788 ms 1.7353 ms 1.7939 ms]
		                          thrpt:  [2.1776 GiB/s 2.2510 GiB/s 2.3268 GiB/s]
		                   change:
		                          time:   [-1.1007% +2.4261% +6.2212%] (p = 0.18 > 0.05)
		                          thrpt:  [-5.8568% -2.3686% +1.1129%]
		                          No change in performance detected.
		  read_full/16.0 MiB      time:   [5.7554 ms 5.9106 ms 6.0651 ms]
		                          thrpt:  [2.5762 GiB/s 2.6436 GiB/s 2.7148 GiB/s]
		                   change:
		                          time:   [+9.0729% +13.626% +18.539%] (p = 0.00 < 0.05)
		                          thrpt:  [-15.639% -11.992% -8.3182%]
		                          Performance has regressed.
		  
		  
		  ```
	- read2
		- ```rust
		  read_full/4.00 KiB      time:   [490.62 us 500.73 us 510.57 us]
		                          thrpt:  [7.6508 MiB/s 7.8011 MiB/s 7.9618 MiB/s]
		                   change:
		                          time:   [-50.327% -49.240% -48.158%] (p = 0.00 < 0.05)
		                          thrpt:  [+92.896% +97.004% +101.32%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  read_full/256 KiB       time:   [558.01 us 570.02 us 581.53 us]
		                          thrpt:  [429.90 MiB/s 438.58 MiB/s 448.02 MiB/s]
		                   change:
		                          time:   [-98.044% -97.998% -97.953%] (p = 0.00 < 0.05)
		                          thrpt:  [+4784.3% +4894.5% +5011.9%]
		                          Performance has improved.
		  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 9.1s, enable flat sampling, or reduce sample count to 50.
		  read_full/4.00 MiB      time:   [1.8975 ms 1.9268 ms 1.9523 ms]
		                          thrpt:  [2.0008 GiB/s 2.0273 GiB/s 2.0586 GiB/s]
		                   change:
		                          time:   [-3.6129% -0.2662% +3.0626%] (p = 0.88 > 0.05)
		                          thrpt:  [-2.9716% +0.2669% +3.7483%]
		                          No change in performance detected.
		  read_full/16.0 MiB      time:   [5.5291 ms 5.6769 ms 5.8211 ms]
		                          thrpt:  [2.6842 GiB/s 2.7524 GiB/s 2.8260 GiB/s]
		                   change:
		                          time:   [-7.4639% -3.9542% -0.5695%] (p = 0.04 < 0.05)
		                          thrpt:  [+0.5728% +4.1170% +8.0659%]
		                          Change within noise threshold.
		  ```
-
- 看着没什么问题，现在需要考虑要暴露怎么样的 API 出去
	- read/write
	- 包装成 Reader 和 Writer？
		- 不支持 AsyncRead，只支持 AsyncBufRead
	- 可以先把 Reader / Writer 之类的抽象先删除
- 需要 bench 一下 pipe 的库
	- https://github.com/yskszk63/tokio-pipe
	- https://github.com/routerify/async-pipe-rs
	- https://docs.rs/tokio/1.17.0/tokio/io/fn.duplex.html
- 考虑到纯内存操作，是不是不需要 async ？
-
- 发现了一个比较大的问题
	- 如果 read 接受一个 Writer 进去的话，用户没法拿到写完之后的结果
	- 但如果接受一个 &mut Writer 的话，又会出现生命周期的问题
-
- channel 也不太行，不是很能工作
-