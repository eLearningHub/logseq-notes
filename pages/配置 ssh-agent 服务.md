- 增加 service 文件， `~/.config/systemd/user/ssh-agent.service`
	- ```
	  [Unit]
	  Description=SSH key agent
	  
	  [Service]
	  Type=simple
	  Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
	  ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK
	  
	  [Install]
	  WantedBy=default.target
	  ```
- 在 `~/.pam_environment` 中指定 AUTH_SOCKET
	- ```
	  SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/ssh-agent.socket"
	  ```
- 启动并配置开机自启
	- ```shell
	  systemctl --user enable ssh-agent
	  systemctl --user start ssh-agent
	  ```
- Reference
	- https://unix.stackexchange.com/questions/339840/how-to-start-and-use-ssh-agent-as-systemd-service