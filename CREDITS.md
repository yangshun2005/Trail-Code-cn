# 致谢

## 语言支持

graft map 的多语言支持由提交语言 Pull Request 的社区贡献者共同塑造。正是这一波一次一种语言的 PR，促成了通用的广度层（每种语言一个 tree-sitter 语法 + `tags.scm`），如今已覆盖其中大部分语言。感谢：

- **@muneebshere**（#38）—— Dart 与 Kotlin。两者现已通过广度层支持。
- **@reinhardtb**（#46）—— C#。
- **@jhouserizer**（#53）与 **@dbianco**（#83）—— Java。
- **@qoole**（#59）—— Rust；（#58）—— PowerShell；（#40）—— tree-sitter 0.25 运行时升级。
- **@williamdes**（#64）—— PHP。
- **@edrethardo**（#67）—— C/C++，以及"对不支持的语言大声报错"的想法（#66）；广度层通过将未知扩展名路由到通用提取器而非跳过它们来解决这一问题。
- **@kapelner**（#69）—— C/C++；（#70）—— R。
- **@CroissantEdit**（#187）—— Lua。
- **@Ansh-Sonkusare**（#159）—— Nix。
- **@shovelmn12**（#130）—— Kotlin，已提升为全保真深度提取。
- **@mattstrayer**（#219）—— Swift。

如果你提交过语言 PR 但未列于此，请开一个 Issue —— 遗漏是疏忽，绝非轻视。