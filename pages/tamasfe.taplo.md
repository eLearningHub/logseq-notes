title:: tamasfe/taplo
type:: [[Project]]
source:: [tamasfe/taplo](https://github.com/tamasfe/taplo)
language:: [[Rust]]

- A [[TOML]] toolkit written in [[Rust]]
-
- 安装
	- ```shell
	  cargo install taplo-cli
	  ```
	- 这个项目提供了 [[Node.js]] 的 wrapper (通过 [[WASM]])，所以也可以通过 [[npm]] 或 [[yarn]] 安装
		- ```shell
		  yarn global add @taplo/cli
		  ```
		- ```shell
		  npm install -g @taplo/cli
		  ```
-
- 配置文件
	- ```toml
	  include = ["Cargo.toml", "**/*.toml"]
	  
	  [formatting]
	  # Align consecutive entries vertically.
	  align_entries = false
	  # Append trailing commas for multi-line arrays.
	  array_trailing_comma = true
	  # Expand arrays to multiple lines that exceed the maximum column width.
	  array_auto_expand = true
	  # Collapse arrays that don't exceed the maximum column width and don't contain comments.
	  array_auto_collapse = true
	  # Omit white space padding from single-line arrays
	  compact_arrays = true
	  # Omit white space padding from the start and end of inline tables.
	  compact_inline_tables = false
	  # Maximum column width in characters, affects array expansion and collapse, this doesn't take whitespace into account.
	  # Note that this is not set in stone, and works on a best-effort basis.
	  column_width = 80
	  # Indent based on tables and arrays of tables and their subtables, subtables out of order are not indented.
	  indent_tables = false
	  # The substring that is used for indentation, should be tabs or spaces (but technically can be anything).
	  indent_string = '  '
	  # Add trailing newline at the end of the file if not present.
	  trailing_newline = true
	  # Alphabetically reorder keys that are not separated by empty lines.
	  reorder_keys = true
	  # Maximum amount of allowed consecutive blank lines. This does not affect the whitespace at the end of the document, as it is always stripped.
	  allowed_blank_lines = 2
	  # Use CRLF for line endings.
	  crlf = false
	  
	  ```
	- 比较有意思的地方在于这个工具对 toml 的语义是有感知的，通过 JSON Schema 支持了 validate
		- [Cargo.toml](https://taplo.tamasfe.dev/configuration/#builtin-schemas) 就提供了官方的 Schema 支持
		- 可以基于这个 schema 来指定更加细节的规则
			- 比如仅对 `Cargo.toml` 中的 `dependencies` 进行排序
			- ```toml
			  [formatting]
			  reorder_keys = false
			  [[rule]]
			  include = ["**/Cargo.toml"]
			  keys = ["dependencies"]
			  [rule.formatting]
			  reorder_keys = true
			  ```
-
- 格式化
	- ```shell
	  taplo fmt
	  ```
- 在 CI 中可以添加 `--check` 参数来检查当前项目是否符合要求
	- ```shell
	  taplo fmt --check
	  ```