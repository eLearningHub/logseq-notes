- 撰写中
	- {{query (and (not (page-property status DONE)) (page-property type Blog))}}
	  query-properties:: [:page]
- 已发布
	- {{query (and (page-property type Blog) (page-property status DONE))}}
	  query-properties:: [:page :link]