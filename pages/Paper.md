- To Read
- {{query (and (not (page-property status DONE)) (page-property type Paper))}}
  query-sort-by:: page
  query-sort-desc:: true
  query-properties:: [:page :conference :doi]
-
- Done
- {{query (and (page-property type Paper) (page-property status DONE))}}
  query-properties:: [:page :conference :doi]
-
- 可参考的资料
	- [Distributed Consensus Reading List](https://github.com/heidihoward/distributed-consensus-reading-list)
	- [[CS 598XU: Reliability of Cloud-Scale Systems]]