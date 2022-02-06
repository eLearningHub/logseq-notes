title:: PushdownDB: Accelerating a DBMS using S3 Computation
type:: [[Paper]]
conference:: [[ICDE'20]]
doi:: [10.1109/ICDE48307.2020.00174](https://doi.org/10.1109/ICDE48307.2020.00174)

- https://arxiv.org/abs/2002.05837
- http://pages.cs.wisc.edu/~yxy/pubs/pushdowndb-icde.pdf
-
- 这篇 paper 主要在研究通过把一些计算下推到 S3 Select 来加速计算
-
- >  Experimentation with a collection of queries including TPC-H queries shows that PushdownDB
  is on average 30% cheaper and 6.7× faster than a baseline that does not use S3 Select.
-
- #question 这篇文章是在对比使用 S3 存储数据，然后使用 S3 Select 和不用的对比？
	-