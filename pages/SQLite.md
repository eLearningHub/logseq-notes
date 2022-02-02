type:: [[Database]]
language:: [[C]]
category:: [[OLTP]], [[SQL]]

-
- 应用
	- SQLite 可以作为一个 embed db 有很多独特的应用场景
	- 比如存储在 [[Cloudflare/Durable Objects]] 中
		- 文章 [Store SQLite in Cloudflare Durable Objects](https://ma.rkusa.st/store-sqlite-in-cloudflare-durable-objects) 中有介绍
			- > A custom SQLite virtual file system and some WASM/WASI compilation magic allow to run SQLite on a Cloudflare Worker and persist it into a Durable Object.
			- > POC source can be found at github.com/rkusa/do-sqlite.
-
- 推荐读物
	- Wesley Aptekar-Cassels: [Consider SQLite](https://blog.wesleyac.com/posts/consider-sqlite)