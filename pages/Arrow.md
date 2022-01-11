alias:: Apache Arrow

- https://arrow.apache.org/
-
- A critical component of Apache Arrow is its in-memory columnar format, a standardized, language-agnostic specification for representing structured, table-like datasets in-memory.
-
- 特点
	- Columnar is Fast
		- ![image.png](../assets/image_1641882058070_0.png){:height 541, :width 776}
	- Standardization Saves
		- ![image.png](../assets/image_1641882072876_0.png)
- 特性
	- 数据相邻
		- Data adjacency for sequential access (scans)
	- O(1) 随机读取
		- O(1) (constant-time) random access
	- 向量化友好
		- SIMD and vectorization-friendly
	-