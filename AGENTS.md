## 本地化（Localization）

所有交互和输出请尽可能使用简体中文。此规则全局有效，除非有例外说明。


### 全局

- 主要以中文表达，包括思考过程、解释、建议、回复等。
- 目录/文件名维持英文习惯（如 README.md、api.md）。
- 技术术语、缩略词（如 JSON、TODO）等由英语环境创造的词，维持英文习惯。
- 字段说明中名称、类型使用英文，但解释说明用中文。
  示例：
  ```text
  - `userId` (string) 用户唯一标识，必填
  - `email` (string) 电子邮箱，需符合邮箱格式，可选
  ```


### 文档&源码

- 若标题必须使用英语，应在末尾附加中文解释，解释放在括号（...）内。
- 文档中的普通说明性内容（如段落、列表等）请使用中文。
- 文档或程序源码中的**注释**也应当使用中文，方便用户理解、审核。

- 程序输出的消息以及日志等沿用英文，如 fmt.Printf(), errors.New(), log.Println() 等的实参。
- 变量、函数、类、对象、方法等名称沿用英语习惯。


<!-- CODEGRAPH_START -->
## CodeGraph

In repositories indexed by CodeGraph (a `.codegraph/` directory exists at the repo root), reach for it BEFORE grep/find or reading files when you need to understand or locate code:

- **MCP tool** (when available): `codegraph_explore` answers most code questions in one call — the relevant symbols' verbatim source plus the call paths between them, including dynamic-dispatch hops grep can't follow. Name a file or symbol in the query to read its current line-numbered source. If it's listed but deferred, load it by name via tool search.
- **Shell** (always works): `codegraph explore "<symbol names or question>"` prints the same output.

If there is no `.codegraph/` directory, skip CodeGraph entirely — indexing is the user's decision.
<!-- CODEGRAPH_END -->
