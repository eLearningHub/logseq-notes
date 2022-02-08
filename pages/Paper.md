- To Read
- {{query (and (page-property type Paper) (not (page-property status DONE)))}}
  query-sort-by:: page
  query-sort-desc:: true
  query-properties:: [:page :type :conference :doi]
-
- {{query (and (page-property type Paper) (page-property status DONE))}}
  query-properties:: [:page :type :conference :doi :status :created-at :updated-at]
-
- 可参考的资料
	- [Distributed Consensus Reading List](https://github.com/heidihoward/distributed-consensus-reading-list)
	- [[CS 598XU: Reliability of Cloud-Scale Systems]]