- Iteration 是来自 Github Project 的概念，用来指代一个固定时间的迭代周期，一般为两周
- 我采用 Iteration 作为一个简单的任务管理模型
-
- {{query  (page-property type Iteration) }}
  query-properties:: [:page :date]
  query-sort-by:: page
  query-sort-desc:: true
-
- 可能的改进
	- 调用 [[Github]] [[GraphQL]] API 获取 task 内容直接展示在 Iteration 里面？
	- 自动创建下一个 Iteration？
		- 从 Github 自动获取是不是更好些？