# 开发日志

真正的日志记录在 issues 页面，那里记录了所有思考、问题以及解决结果：

* <https://github.com/os-checker/os-checker/issues>
* <https://github.com/os-checker/os-checker.github.io/issues>


每周开发总结：

- [第 1-2 周]：设计和解析配置文件；实现 fmt 和 clippy 检查
- [第 3 周]：实现主页诊断数量表
- [第 4 周]：实现问题文件树，展示所有仓库的原始检查结果
- [第 5 周]：重构 JSON 输出；把数据处理迁移到 database 仓库
- [第 6 周]：支持编译目标搜索；WebUI 添加编译目标下拉框
- [第 7 周]：集成 lockbud； 添加 Cargo 检查报告；WebUI 增加 All-Targets 统计
- [第 8 周]：成功检查 88 个 kern-crates 相关的仓库；使用 Rust 重构 jq 数据处理；实现仓库级别的问题文件树
- [第 9 周]：处理遗留出错的 18 个仓库；尝试集成 mirai
- [第 10-11 周]：WebUI 增加统计图；实现数据库缓存
- [第 12 周]：缓存和展示编译目标明细信息
- [第 13 周]：编写 WebUI 使用说明；创建 os-checker book 文档；集成 cargo-audit
- [第 14 周]：集成 RAP 和 cargo-outdated
- [第 15 周]：WebUI 增加 Github Actions 页面；给 RAP 提交重构 PR；集成 cargo-geiger；解析 package 和测例信息
- [第 16 周]：WebUI 增加 info 页面；集成 Rudra；部署在线文档
- [第 17 周]：WebUI 添加 repos 页面、实现路由查询；在 kern-crates 组织部署 OS 组件库的 rustdoc 文档和 os-checker 的 WebUI；编写 os-checker 组件化目标文档
- [第 18 周]：调整 info 页面 package 的范围；给 RAP 提交修复和新功能；更新 os-checker book 文档；查看 Rust Formal Methods WG/IG 相关资料；WebUI 细微调整
- [第 19 周]：实现 os-checker 工具集的 Docker 镜像和 Github Action Workflows；在 package 信息表上新增字段；发布所有插件 crates 到 crates.io
- [第 20 周]：重写 kern-crates/.github；WebUI 主页与 info 页面互换；集成 cargo-semver-checks；编写自动化部署文档；阅读/翻译
- [第 22 周]：初步集成 Miri；WebUI 增加 testcases 页面，展示测试和 Miri 详情
- [第 23 周]：查看全部有诊断的 143 个仓库的诊断信息，分析和记录产生诊断的原因，并修正部分仓库的 targets 配置
- [第 24 周]：os-checker 发布 v0.5.0；一些琐事和未完成的事情；年终总结/展望
- [第 25 周]：给 OS 组件库提交诊断修复；修改 kern-crates/.github 的模板
- [第 26 周]：给 OS 组件库提交诊断修复；os-checker 更新检查工具和工具链
- [第 27 周]：kern-crates 组织添加和更新仓库的策略、处理重命名的外部仓库；给 OS 组件库提交诊断修复
- [第 28 周]：编写 os-checker PPT（面向使用者）；os-checker 在 JSON 配置文件中支持指定 features
- [第 29-30 周]：初步重构诊断详情页面；PPT 演讲准备完毕
- [第 31 周]：file-tree 页面添加筛选项来交互式展示诊断详情
- [第 32 周]：修改 PPT；file-tree 页面支持路由参数查询；testcases 页面添加筛选下拉框查询
- [第 33 周]：进行 os-checker 报告；plugin-cargo 支持缓存；WebUI 改进；文档添加《工作原理》一章
- [第 34 周]: 研究 Charon、Rudra 和 Charon-Rudra；os-checker 更新检查工具及其工具链
- [第 35 周]: 给 Charon 提交重构 CLI & 修复 PRs；研究 Rudra 源代码
- [第 36 周]: 给 Charon 提交 PRs；使用 Charon 数据源复现 Rudra 的 Unsafe Destructor 分析；阅读和记录 SendSyncVariance 分析
- [第 37 周]: Charon-Rudra 使用 Charon 数据源复现 Rudra 的 Send / Sync 分析；一些计划
- [第 38 周]: 阅读 Kani 代码和文档；os-checker 小修复
- [第 39 周]: 创建 distributed-verification 仓库，从每个 `#[kani::proof]` 入口获取所有调用，计算稳定的 hash 值
- [第 40 周]: distributed-verification 测试；Rust Workshop 提纲；term-rustdoc 更新
- [第 41 周]: os-checker 许可从 MIT 改为 GPL-3.0 OR MulanPubL；distributed-verification 处理 kani-list.json 和 libcore proofs
- [第 42 周]: os-checker 与 Chain-Fox 合作；os-checker 修复和添加新功能；distributed-verification 支持新参数
- [第 43 周]: os-checker 新增配置字段和命令行参数；给 tag-std 编写基于 Rust 编译器驱动的 demo
- [第 44 周]: io-uring 异步运行时基准测试；distributed-verification 增加 --stat 参数，统计标注 kani 属性的函数；阅读 RustWeek - All Hands 会议记录
- [第 45 周]: tag-std 实现基本的安全属性标注语法和 safety 文档生成；os-checker 重新检查所有仓库
- [第 46 周]: tag-std - 实现本地安全属性 discharge 检测；AsyncOS.github.io - 异步操作系统设计文档部署；创建 hugo-book-template 模板仓库
- [第 47 周]: tag-std 实现跨 crates 自定义安全属性的 discharges 检测、让 tag-std 在 verify-rust-std 上工作；完成一些训练营任务；准备下周的 Workshop
- [第 48 周]: 参加 Rust Workshop；tag-std 支持内置安全属性的 discharges 检查
- [第 49 周]: distributed-verification 重构：搜集所有可达函数，而不是局限于 proofs
- [第 50 周]: os-checker 发布 v0.8.0：集成 AtomVChecker 和 Udeps 检查；tag-std 在 Rust for Linux 代码库进行编译和检查；distributed-verification 缓存函数信息到 SQLite 数据库文件并修复测试
- [第 51 周]: tag-std 在 Rust for Linux 上运行安全属性标注的代码；distributed-verification 同步 verify-rust-std
- [第 52 周]: 将 tag-std 应用于 Asterinas OS 安全属性标注；kern-crates 发布 v0.4.0，只检查清单内的仓库
- [第 53 周]: tag-std：编写 pre-RFC Safety Property System
- [第 54 周]: 与社区讨论 Pre-RFC: Safety Property System；tag-std：实现新语法 `#[safety { SP1, SP2: "shared reasons" }]`，并支持动态属性定义
- [第 55 周]: 向 Rust 项目正式提交 RFC《Safety Tags》并广泛讨论；tag-std 配置文件相关功能更新和修复
- [第 56 周]: distributed-verification 实现初步的 Kani proofs 基础信息和条件筛选 UI；RFC: Safety Tags 社区讨论
- [第 57 周]: tag-std 实现 any 标签、部署安全文档、打开星绽 RFC 讨论；distributed-verification：修复 JSON 合并问题、新增 UI 功能；verify-rust-std：开始与上游对接
- [第 58 周]: distributed-verification：UI 新增 proof 和函数源码展示、推迟 GSoC 项目结束时间到 11 月；verify-rust-std & kani：查明编译失败问题；RFC: Safety Tags 继续社区讨论
- [第 59 周]: os-checker：准备 9 月杭州 RustChinaConf 2025 演讲；tag-std：将 verify-rust-std 纳入 CI，并提供 RAPx 接口；distributed-verification：UI 更新；旁听 Asterinas 和 Axvisor 周会
- [第 60 周]: os-checker book 新增三个页面，关于工作进展和贡献点；distributed-verification 支持 libcore 之外的 Kani 证明；更新 Safety Tags RFC
- [第 62 周]: os-checker：在 RustChinaConf2025 演讲 | 手记 | 被旋武社区收录；与星绽开发人员就安全属性标注问题进行答疑和讨论；初步实现 safety-lsp
- [第 63 周]: safety-lsp 实现安全属性补全和 VSCode 插件功能；准备 os-checker 10 月 4 日线上分享 PPT；尝试复现星绽 KernMiri
- [第 64-65 周]: os-checker 10 月 4 日线上报告；safety-tool 发布 v0.4.0；星绽 KernMiri 源码阅读和交流
- [第 66 周]: Distributed-Verification WebUI 更新：增加 Kani 验证时间和数量分布图、支持 autoharnesses
- [第 67 周]: 准备分享《编写 Rust 静态分析工具》；tag-std 发布 v0.4.1；distributed-verification 更新
- [第 68 周]: Rust 编译器训练营课程分享《编写 Rust 静态分析工具》；distributed-verification 更新
- [第 69 周]: os-checker：更新和修复 Miri 测例运行与结果、发布 plugin-cargo v0.1.7；book 更新：增加 KernMiri 和 Miri 笔记；distributed-verification：提交 GSoC2025 结题报告
- [第 70 周]: 修复 Miri 报告的 ArceOS linked_list_r4l 问题代码，并对照当前 Rust for Linux 的链表实现
- [第 71 周]: ArceOS 修复和问题排查；os-checker 更新
- [第 72 周]: 星绽 RFC - 安全属性标注；os-checker 新增检查星绽；safety-tags 添加统计输出 JSON
- [第 73 周]: safety-tool 生成 Rust for Linux 和星绽的安全属性统计信息；准备 14 号演讲《Rust 安全属性标注》slides
- [第 74 周]: 演讲《Rust 安全属性标注》；safety-tool 更新
- [第 75 周]: 发现和排查问题：ArceOS、星绽和 RAPx；更新 book：新增 Miri 论文阅读笔记
- [第 76 周]: 《os-checker 2025 年度总结》并发布 v0.8.1；修复 kern-crates 同步问题
- [第 77 周]: unsafety-propagation-graph 可视化不安全代码传播图
- [第 78 周]: unsafety-propagation-graph 实现导航栏、渲染安全属性节点
- [第 79 周]: unsafety-propagation-graph 新增标准库、星绽 ostd 库展示；调用图功能增强与样式美化
- [第 80 周]: unsafety-propagation-graph：字段访问关联被调用者；URL 直达和路由参数交互；导航增强（模块树、文档链接等）；安全属性统计图
- [第 81 周]: unsafety-propagation-graph 添加 ADT 外生审计函数列表、重构调用图、添加 README；调查 Rust 动态链接与异步共享库方案
- [第 82 周]: 《Rust 动态链接与异步共享库》笔记；KMiri：整合仓库、在星绽 OSDK 添加 miri 子命令
- [第 83 周]: KMiri：更新星绽 ostd 和 Miri 代码；KLint：调查 not_using_prelude 和 Atomic Context Violations 检测

[第 1-2 周]: https://github.com/os-checker/os-checker/blob/3fdf88db57403949f95c3034608481d64db80764/assets/development-logs.md
[第 3 周]: https://github.com/os-checker/os-checker/discussions/15
[第 4 周]: https://github.com/os-checker/os-checker/discussions/20
[第 5 周]: https://github.com/os-checker/os-checker/discussions/24
[第 6 周]: https://github.com/os-checker/os-checker/discussions/32
[第 7 周]: https://github.com/os-checker/os-checker/discussions/41
[第 8 周]: https://github.com/os-checker/os-checker/discussions/66
[第 9 周]: https://github.com/os-checker/os-checker/discussions/90
[第 10-11 周]: https://github.com/os-checker/os-checker/discussions/104
[第 12 周]: https://github.com/os-checker/os-checker/discussions/121
[第 13 周]: https://github.com/os-checker/os-checker/discussions/136
[第 14 周]: https://github.com/os-checker/os-checker/discussions/145
[第 15 周]: https://github.com/os-checker/os-checker/discussions/159
[第 16 周]: https://github.com/os-checker/os-checker/discussions/163
[第 17 周]: https://github.com/os-checker/os-checker/discussions/164
[第 18 周]: https://github.com/os-checker/os-checker/discussions/170
[第 19 周]: https://github.com/os-checker/os-checker/discussions/185
[第 20 周]: https://github.com/os-checker/os-checker/discussions/189
[第 22 周]: https://github.com/os-checker/os-checker/discussions/193
[第 23 周]: https://github.com/os-checker/os-checker/discussions/225
[第 24 周]: https://github.com/os-checker/os-checker/discussions/249
[第 25 周]: https://github.com/os-checker/os-checker/discussions/255
[第 26 周]: https://github.com/os-checker/os-checker/discussions/263
[第 27 周]: https://github.com/os-checker/os-checker/discussions/265
[第 28 周]: https://github.com/os-checker/os-checker/discussions/270
[第 29-30 周]: https://github.com/os-checker/os-checker/discussions/278
[第 31 周]: https://github.com/os-checker/os-checker/discussions/284
[第 32 周]: https://github.com/os-checker/os-checker/discussions/287
[第 33 周]: https://github.com/os-checker/os-checker/discussions/291
[第 34 周]: https://github.com/os-checker/os-checker/discussions/301
[第 35 周]: https://github.com/os-checker/os-checker/discussions/302
[第 36 周]: https://github.com/os-checker/os-checker/discussions/303
[第 37 周]: https://github.com/os-checker/os-checker/discussions/304
[第 38 周]: https://github.com/os-checker/os-checker/discussions/308
[第 39 周]: https://github.com/os-checker/os-checker/discussions/309
[第 40 周]: https://github.com/os-checker/os-checker/discussions/310
[第 41 周]: https://github.com/os-checker/os-checker/discussions/316
[第 42 周]: https://github.com/os-checker/os-checker/discussions/339
[第 43 周]: https://github.com/os-checker/os-checker/discussions/357
[第 44 周]: https://github.com/os-checker/os-checker/discussions/359
[第 45 周]: https://github.com/os-checker/os-checker/discussions/364
[第 46 周]: https://github.com/os-checker/os-checker/discussions/365
[第 47 周]: https://github.com/os-checker/os-checker/discussions/367
[第 48 周]: https://github.com/os-checker/os-checker/discussions/368
[第 49 周]: https://github.com/os-checker/os-checker/discussions/369
[第 50 周]: https://github.com/os-checker/os-checker/discussions/378
[第 51 周]: https://github.com/os-checker/os-checker/discussions/379
[第 52 周]: https://github.com/os-checker/os-checker/discussions/381
[第 53 周]: https://github.com/os-checker/os-checker/discussions/382
[第 54 周]: https://github.com/os-checker/os-checker/discussions/383
[第 55 周]: https://github.com/os-checker/os-checker/discussions/384
[第 56 周]: https://github.com/os-checker/os-checker/discussions/385
[第 57 周]: https://github.com/os-checker/os-checker/discussions/386
[第 58 周]: https://github.com/os-checker/os-checker/discussions/387
[第 59 周]: https://github.com/os-checker/os-checker/discussions/388
[第 60 周]: https://github.com/os-checker/os-checker/discussions/389
[第 62 周]: https://github.com/os-checker/os-checker/discussions/389
[第 63 周]: https://github.com/os-checker/os-checker/discussions/391
[第 64-65 周]: https://github.com/os-checker/os-checker/discussions/392
[第 66 周]: https://github.com/os-checker/os-checker/discussions/393
[第 67 周]: https://github.com/os-checker/os-checker/discussions/394
[第 68 周]: https://github.com/os-checker/os-checker/discussions/395
[第 69 周]: https://github.com/orgs/os-checker/discussions/400
[第 70 周]: https://github.com/orgs/os-checker/discussions/401
[第 71 周]: https://github.com/orgs/os-checker/discussions/408
[第 72 周]: https://github.com/orgs/os-checker/discussions/413
[第 73 周]: https://github.com/orgs/os-checker/discussions/414
[第 74 周]: https://github.com/orgs/os-checker/discussions/415
[第 75 周]: https://github.com/orgs/os-checker/discussions/416
[第 76 周]: https://github.com/orgs/os-checker/discussions/425
[第 77 周]: https://github.com/orgs/os-checker/discussions/426
[第 78 周]: https://github.com/orgs/os-checker/discussions/427
[第 79 周]: https://github.com/orgs/os-checker/discussions/428
[第 80 周]: https://github.com/orgs/os-checker/discussions/429
[第 81 周]: https://github.com/orgs/os-checker/discussions/430
[第 82 周]: https://github.com/orgs/os-checker/discussions/431
[第 83 周]: https://github.com/orgs/os-checker/discussions/432
