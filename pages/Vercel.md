type:: [[Product]]
features:: [[Web Host]]

- 域名重定向
	- 将 https://work.xuanwo.io 重定向到 https://github.com/users/Xuanwo/projects/2 可以这样
	- 创建一个项目，增加 `vercel.json`
	- ```json
	  {
	    "redirects": [
	      { "source": "/", "destination": "https://github.com/users/Xuanwo/projects/2" }
	    ]
	  }
	  ```
	- 在 vercel 创建一个不需要构建的项目即可