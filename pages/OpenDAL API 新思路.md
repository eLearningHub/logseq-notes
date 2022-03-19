- ```rust
  async fn read(&self, w: BoxedAsyncWrite, args: &OpRead) -> Result<usize>
  ```
-
- 其实可以考虑底层支持多种 API
	- ```rust
	  async fn read(&self, w: BoxedAsyncWrite, args: &OpRead) -> Result<usize>
	  ```
	- ```rust
	  async fn read(&self, sender: tokio::sync::mpsc::Sender<bytes>, args: &OpRead) -> Result<usize>
	  ```
	- 这样会导致 service 实现的复杂度高很多
		- 需要 bench 一下 channel 发送 bytes 的开销到底有多大
-
- Reader2 性能
	- 基础(当前的main)
		- ```rust
		  s3/read                 time:   [4.9304 ms 5.0706 ms 5.2110 ms]
		                          thrpt:  [2.9985 GiB/s 3.0815 GiB/s 3.1691 GiB/s]
		                   change:
		                          time:   [-14.078% -10.663% -6.5827%] (p = 0.00 < 0.05)
		                          thrpt:  [+7.0466% +11.936% +16.384%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  s3/read2                time:   [5.9507 ms 6.0480 ms 6.1428 ms]
		                          thrpt:  [2.5436 GiB/s 2.5835 GiB/s 2.6257 GiB/s]
		                   change:
		                          time:   [+2.4073% +6.0430% +9.8358%] (p = 0.00 < 0.05)
		                          thrpt:  [-8.9550% -5.6986% -2.3507%]
		                          Performance has regressed.
		  Found 2 outliers among 100 measurements (2.00%)
		    2 (2.00%) low mild
		  
		  fs not set, ignore
		  s3_parallel/parallel_range_read_2
		                          time:   [3.2178 ms 3.2730 ms 3.3382 ms]
		                          thrpt:  [4.6806 GiB/s 4.7739 GiB/s 4.8557 GiB/s]
		                   change:
		                          time:   [-3.6668% -0.7349% +2.3554%] (p = 0.64 > 0.05)
		                          thrpt:  [-2.3012% +0.7404% +3.8064%]
		                          No change in performance detected.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high severe
		  s3_parallel/parallel_range_read_4
		                          time:   [4.4657 ms 4.5850 ms 4.7094 ms]
		                          thrpt:  [6.6357 GiB/s 6.8158 GiB/s 6.9978 GiB/s]
		                   change:
		                          time:   [+6.1255% +9.6108% +13.633%] (p = 0.00 < 0.05)
		                          thrpt:  [-11.998% -8.7681% -5.7719%]
		                          Performance has regressed.
		  s3_parallel/parallel_range_read_8
		                          time:   [6.3954 ms 6.6064 ms 6.8245 ms]
		                          thrpt:  [9.1581 GiB/s 9.4605 GiB/s 9.7726 GiB/s]
		                   change:
		                          time:   [-25.968% -21.143% -15.792%] (p = 0.00 < 0.05)
		                          thrpt:  [+18.754% +26.813% +35.076%]
		                          Performance has improved.
		  Found 3 outliers among 100 measurements (3.00%)
		    3 (3.00%) high mild
		  s3_parallel/parallel_range_read_16
		                          time:   [17.556 ms 18.165 ms 18.795 ms]
		                          thrpt:  [6.6508 GiB/s 6.8813 GiB/s 7.1200 GiB/s]
		                   change:
		                          time:   [-19.272% -14.890% -10.318%] (p = 0.00 < 0.05)
		                          thrpt:  [+11.505% +17.496% +23.873%]
		                          Performance has improved.
		  Found 3 outliers among 100 measurements (3.00%)
		    3 (3.00%) high mild
		  
		  ```
	- 使用了一个内部的 channel
		- ```rust
		  s3/read2                time:   [5.7093 ms 5.7913 ms 5.8743 ms]
		                          thrpt:  [2.6599 GiB/s 2.6980 GiB/s 2.7368 GiB/s]
		                   change:
		                          time:   [-8.9355% -6.9789% -4.8184%] (p = 0.00 < 0.05)
		                          thrpt:  [+5.0623% +7.5025% +9.8122%]
		                          Performance has improved.
		  Found 2 outliers among 100 measurements (2.00%)
		    2 (2.00%) high mild
		  
		  fs not set, ignore
		  s3_parallel/parallel_range_read_2
		                          time:   [3.4875 ms 3.5802 ms 3.6754 ms]
		                          thrpt:  [4.2513 GiB/s 4.3643 GiB/s 4.4802 GiB/s]
		                   change:
		                          time:   [-19.928% -17.502% -14.545%] (p = 0.00 < 0.05)
		                          thrpt:  [+17.021% +21.215% +24.887%]
		                          Performance has improved.
		  Found 3 outliers among 100 measurements (3.00%)
		    3 (3.00%) high mild
		  s3_parallel/parallel_range_read_4
		                          time:   [4.5647 ms 4.7198 ms 4.8762 ms]
		                          thrpt:  [6.4087 GiB/s 6.6211 GiB/s 6.8460 GiB/s]
		                   change:
		                          time:   [-42.785% -40.591% -38.407%] (p = 0.00 < 0.05)
		                          thrpt:  [+62.357% +68.325% +74.780%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  s3_parallel/parallel_range_read_8
		                          time:   [8.3315 ms 8.6573 ms 9.0001 ms]
		                          thrpt:  [6.9444 GiB/s 7.2194 GiB/s 7.5016 GiB/s]
		                   change:
		                          time:   [-56.363% -54.543% -52.529%] (p = 0.00 < 0.05)
		                          thrpt:  [+110.66% +119.99% +129.16%]
		                          Performance has improved.
		  s3_parallel/parallel_range_read_16
		                          time:   [21.719 ms 21.968 ms 22.212 ms]
		                          thrpt:  [5.6276 GiB/s 5.6900 GiB/s 5.7553 GiB/s]
		                   change:
		                          time:   [-46.328% -45.428% -44.540%] (p = 0.00 < 0.05)
		                          thrpt:  [+80.311% +83.244% +86.318%]
		                          Performance has improved.
		  
		  ```
	- 把 channel 的 buffer 数量加大
		- 有不同比例的性能提升
		- ```rust
		  s3/read2                time:   [5.5278 ms 5.7034 ms 5.8777 ms]
		                          thrpt:  [2.6583 GiB/s 2.7396 GiB/s 2.8266 GiB/s]
		                   change:
		                          time:   [-4.6765% -1.5186% +1.6649%] (p = 0.38 > 0.05)
		                          thrpt:  [-1.6377% +1.5420% +4.9059%]
		                          No change in performance detected.
		  
		  fs not set, ignore
		  s3_parallel/parallel_range_read_2
		                          time:   [3.2186 ms 3.2973 ms 3.3747 ms]
		                          thrpt:  [4.6301 GiB/s 4.7388 GiB/s 4.8546 GiB/s]
		                   change:
		                          time:   [-11.214% -7.9038% -4.7505%] (p = 0.00 < 0.05)
		                          thrpt:  [+4.9874% +8.5821% +12.630%]
		                          Performance has improved.
		  s3_parallel/parallel_range_read_4
		                          time:   [4.0911 ms 4.1829 ms 4.2774 ms]
		                          thrpt:  [7.3058 GiB/s 7.4708 GiB/s 7.6385 GiB/s]
		                   change:
		                          time:   [-14.840% -11.374% -7.7887%] (p = 0.00 < 0.05)
		                          thrpt:  [+8.4465% +12.833% +17.427%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  s3_parallel/parallel_range_read_8
		                          time:   [7.9268 ms 8.3778 ms 8.8517 ms]
		                          thrpt:  [7.0608 GiB/s 7.4602 GiB/s 7.8847 GiB/s]
		                   change:
		                          time:   [-9.2036% -3.2287% +3.8163%] (p = 0.34 > 0.05)
		                          thrpt:  [-3.6760% +3.3364% +10.137%]
		                          No change in performance detected.
		  s3_parallel/parallel_range_read_16
		                          time:   [20.495 ms 21.343 ms 22.179 ms]
		                          thrpt:  [5.6359 GiB/s 5.8566 GiB/s 6.0990 GiB/s]
		                   change:
		                          time:   [-7.1786% -2.8451% +1.2502%] (p = 0.17 > 0.05)
		                          thrpt:  [-1.2347% +2.9284% +7.7338%]
		                          No change in performance detected.
		  
		  
		  ```
	- 在高并发的情况下看起来还是原来的实现更有优势
	- 感觉内部的实现好像没有什么搞头？
- ---
- Update at [[2022-03-20]]
	-
- 最底层的实现使用
	- ```rust
	  async fn read(&self, args: &OpRead, buf: tokio::io::ReadBuf)
	  ```
- 在这个基础上可以实现
	- ```rust
	  async fn read(&self, args: &OpRead, w: Box<dyn AsyncWrite>)
	  ```
- 以及
	- ```rust
	  async fn read(&self, args: &OpRead) -> Result<OwnedReader>
	  ```
	- OwnedReader 会提供两个接口
		- 一个是 async fetch ，另一个是 sync 的 buffer() -> &[u8]
		- 用户可以将 fetch 调度到 async runtime 上，而真正去读取数据的操作是完全同步的
	- 作为 AccessorReadExt ？
- 还是可以在外层包装 reader 的
-
- 感觉可以搞一搞
- read (`async fn read(&self, args: &OpRead) -> Result<BoxedAsyncReader>`)
	- ```rust
	  read_full/4.00 KiB      time:   [501.63 us 510.70 us 519.31 us]
	                          thrpt:  [7.5220 MiB/s 7.6488 MiB/s 7.7871 MiB/s]
	                   change:
	                          time:   [-14.730% -12.242% -9.6087%] (p = 0.00 < 0.05)
	                          thrpt:  [+10.630% +13.949% +17.275%]
	                          Performance has improved.
	  read_full/256 KiB       time:   [574.83 us 585.27 us 595.59 us]
	                          thrpt:  [419.75 MiB/s 427.15 MiB/s 434.91 MiB/s]
	                   change:
	                          time:   [-18.059% -15.562% -12.993%] (p = 0.00 < 0.05)
	                          thrpt:  [+14.933% +18.430% +22.040%]
	                          Performance has improved.
	  Found 2 outliers among 100 measurements (2.00%)
	    1 (1.00%) low mild
	    1 (1.00%) high mild
	  read_full/4.00 MiB      time:   [2.7627 ms 2.7974 ms 2.8320 ms]
	                          thrpt:  [1.3793 GiB/s 1.3964 GiB/s 1.4139 GiB/s]
	                   change:
	                          time:   [+67.643% +70.017% +72.427%] (p = 0.00 < 0.05)
	                          thrpt:  [-42.004% -41.182% -40.349%]
	                          Performance has regressed.
	  Found 2 outliers among 100 measurements (2.00%)
	    1 (1.00%) low mild
	    1 (1.00%) high mild
	  read_full/16.0 MiB      time:   [11.173 ms 11.276 ms 11.377 ms]
	                          thrpt:  [1.3733 GiB/s 1.3856 GiB/s 1.3985 GiB/s]
	                   change:
	                          time:   [+110.02% +112.29% +114.50%] (p = 0.00 < 0.05)
	                          thrpt:  [-53.381% -52.894% -52.386%]
	                          Performance has regressed.
	  
	  
	  ```
- read2 (`async fn read2(&self, args: &OpRead, buf: &mut tokio::io::ReadBuf)`)
	- ```rust
	  read_full/4.00 KiB      time:   [539.70 us 553.03 us 567.10 us]
	                          thrpt:  [6.8882 MiB/s 7.0633 MiB/s 7.2378 MiB/s]
	                   change:
	                          time:   [+13.907% +17.540% +20.795%] (p = 0.00 < 0.05)
	                          thrpt:  [-17.215% -14.923% -12.209%]
	                          Performance has regressed.
	  Found 1 outliers among 100 measurements (1.00%)
	    1 (1.00%) low mild
	  read_full/256 KiB       time:   [601.84 us 613.62 us 625.22 us]
	                          thrpt:  [399.86 MiB/s 407.42 MiB/s 415.39 MiB/s]
	                   change:
	                          time:   [-8.2457% -5.6658% -2.6203%] (p = 0.00 < 0.05)
	                          thrpt:  [+2.6909% +6.0061% +8.9867%]
	                          Performance has improved.
	  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
	  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 8.4s, enable flat sampling, or reduce sample count to 50.
	  read_full/4.00 MiB      time:   [1.7123 ms 1.7257 ms 1.7382 ms]
	                          thrpt:  [2.2473 GiB/s 2.2636 GiB/s 2.2812 GiB/s]
	                   change:
	                          time:   [+4.5913% +5.4065% +6.2071%] (p = 0.00 < 0.05)
	                          thrpt:  [-5.8443% -5.1291% -4.3897%]
	                          Performance has regressed.
	  Found 1 outliers among 100 measurements (1.00%)
	    1 (1.00%) high mild
	  read_full/16.0 MiB      time:   [5.4221 ms 5.4382 ms 5.4552 ms]
	                          thrpt:  [2.8642 GiB/s 2.8732 GiB/s 2.8817 GiB/s]
	                   change:
	                          time:   [-0.7715% -0.3979% -0.0334%] (p = 0.04 < 0.05)
	                          thrpt:  [+0.0334% +0.3995% +0.7775%]
	                          Change within noise threshold.
	  
	  ```
	- 4MB 以及 16MB 的 case 上有奇怪的双倍性能提升，怀疑是测试方式有问题导致的
		- 是因为减少了一次不必要的内存复制吗？
- 尝试直接使用 `&mut [8]`: (`async fn read2(&self, args: &OpRead, mut buf: &mut [u8])`)
	- ```rust
	  read_full/4.00 KiB      time:   [535.73 us 550.49 us 564.94 us]
	                          thrpt:  [6.9145 MiB/s 7.0959 MiB/s 7.2915 MiB/s]
	                   change:
	                          time:   [-3.4323% -0.4607% +2.4127%] (p = 0.77 > 0.05)
	                          thrpt:  [-2.3558% +0.4629% +3.5543%]
	                          No change in performance detected.
	  Found 2 outliers among 100 measurements (2.00%)
	    1 (1.00%) low mild
	    1 (1.00%) high mild
	  read_full/256 KiB       time:   [661.80 us 682.63 us 702.10 us]
	                          thrpt:  [356.07 MiB/s 366.23 MiB/s 377.76 MiB/s]
	                   change:
	                          time:   [+12.137% +15.556% +19.508%] (p = 0.00 < 0.05)
	                          thrpt:  [-16.323% -13.462% -10.824%]
	                          Performance has regressed.
	  Found 2 outliers among 100 measurements (2.00%)
	    1 (1.00%) low mild
	    1 (1.00%) high mild
	  Benchmarking read_full/4.00 MiB: Warming up for 3.0000 s
	  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 8.4s, enable flat sampling, or reduce sample count to 50.
	  read_full/4.00 MiB      time:   [1.6280 ms 1.6425 ms 1.6571 ms]
	                          thrpt:  [2.3573 GiB/s 2.3782 GiB/s 2.3995 GiB/s]
	                   change:
	                          time:   [-4.7335% -3.8071% -2.8355%] (p = 0.00 < 0.05)
	                          thrpt:  [+2.9183% +3.9578% +4.9686%]
	                          Performance has improved.
	  Found 1 outliers among 100 measurements (1.00%)
	    1 (1.00%) high mild
	  read_full/16.0 MiB      time:   [5.2822 ms 5.3119 ms 5.3416 ms]
	                          thrpt:  [2.9251 GiB/s 2.9415 GiB/s 2.9580 GiB/s]
	                   change:
	                          time:   [-2.9437% -2.3224% -1.6819%] (p = 0.00 < 0.05)
	                          thrpt:  [+1.7107% +2.3776% +3.0330%]
	                          Performance has improved.
	  
	  ```
	- 能更快一点点
- 为什么比返回一个 BoxedAsyncRead 快呢？又为什么返回一个 BoxedAsyncRead 在小数据量的情况下会更快呢？
	- ```rust
	  let mut buf = buf.clone();
	  let mut r = op.object(path).limited_reader(size.bytes());
	  r.read_exact(&mut buf).await.unwrap();
	  ```
	- ```rust
	  let mut buf = buf.clone();
	  op.object(path).read(0, size.bytes(), &mut buf).await;
	  ```
- 发出去的请求应该是一样的
- 之前的实现会重复创建 feature，修复之后
	- ```rust
	  ```