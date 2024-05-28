```text
1. Commit message 的格式规范

每次提交，Commit message 都包括三个部分：Header，Body 和 Footer。

<type>: <subject> 
// 空一行 
<body> 
// 空一行 
<footer>

其中，Header 是必需的，Body 和 Footer 可以省略。
footer：是issue连接或者wiki

1.1 Message Subject（Header）

Header部分只有一行，包括三个字段：type（必需）、和subject（必需），subject之前一定要留一个空格。

1）type

type用于说明 commit 的类别，如：

* feat: 新功能
* fix: 修改bug
* docs: 修改文档
* style: 格式化代码结构（不影响代码运行的变动)
* perf: 性能优化
* refactor: 重构（即不是新增功能，也不是修改bug的代码变动，比如重命名变量）
* chore: 构建过程或辅助工具的变动（不会影响代码运行）
* revert:如果当前 commit 用于撤销以前的 commit，则必须以revert:开头，后面跟着被撤销 Commit 的 Header
   example: revert: feat: add 'graphiteWidth' option 

如果type为feat和fix，则该 commit 将肯定出现在 Change log 之中。
其他情况（docs、chore、style、refactor、test）由你决定，要不要放入 Change log，建议是不要

2）subject

subject是 commit 目的的简短描述，不超过50个字符。
```