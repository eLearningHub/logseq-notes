title:: 2022-11: 新轮子 reqsign
type:: [[Blog]]

- 这周主要的时间都在搓新轮子 [reqsign](https://github.com/Xuanwo/reqsign)，用于对用户的请求进行签名，使得用户不再需要依赖完整的 SDK，我将其概括为 `Signing API requests without effort`。今天这期周报就主要聊聊为什么要造这个轮子，以及适合使用它的场景。
-
- 背景
	- 开发云上服务不可避免的会使用 SDK，它帮助用户处理认证，构造请求，解析响应等任务，使得用户不需要关心服务内部的细节。
-