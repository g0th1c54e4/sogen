# 仓库指南

## 项目概览

Sogen 是一个 C++20 Windows 用户空间模拟器。
它产出一个 `analyzer` 二进制文件，用于分析和模拟 Windows 二进制程序。
细节和命令行选项见 `src/analyzer/main.cpp`

## 构建

开发期间快速迭代，构建 release 预设：

`cmake --build --preset=release`

功能完全实现后，务必构建包含 clang-tidy 的 tidy 配置。
它很慢，所以只在最后用：

`cmake --build --preset=tidy`

## 冒烟测试

在所构建预设的目录下用 cmd 执行 `analyzer.exe --network live -s test-sample.exe` 做冒烟测试。
例如：`cmd /c "cd build\release\artifacts\ && analyzer.exe --network live -s test-sample.exe"`
该样本会解析 `google.com`，需要 `--network live`。

其他应用也可以在模拟器中执行：
`cmd /c "cd build\release\artifacts\ && analyzer.exe -s path/to/binary.exe arg1 arg2 ..."`

## 测试

测试不是越多越好。有用时才加测试；没有测试好过糟糕的测试。不要仅为提高覆盖率而加测试。

- 好的测试用客观的外部基准来钉住行为：Windows/NT 语义、客体可见结果、公开 API、ABI 与文件格式、架构不变量。把它当作指导原则并运用判断力：超出此范围测试仍可能有意义。
- 不要钉住开发者自选的呈现形式，如日志与诊断消息的措辞或内容、输出格式，除非该输出是显式的稳定接口或被另一个组件消费。
- 避免几乎无价值的测试：同义反复、照抄实现的测试、钉住单一任意情形中偶然返回值的测试、纯接线或转发检查。
- 优先单元测试（Google Test）。它们轻量且快得多。仅当行为真正横跨模拟器子系统且无法在单元层有效检验时，才使用端到端 `test-sample.exe`（在模拟器内运行）。CI 是有限资源。
- 测试必须确定且高效。不要无谓的 sleep 或固定延迟，不做依赖时序的断言。需要等待时，等待真实条件。

## 代码注释

默认**不写注释**。代码应通过命名和结构自我解释：职责单一的小型自包含函数、精确的变量与函数命名、善用类、RAII、容器与抽象。代码应该像书一样可读。

注释仅在承载*无法从代码中恢复*的信息、且称职的开发者读本文件时会因此犯错时才正当。实践中即：

- 非显然的外部约束：Windows/NT 行为、无文档的结构体布局、迫使代码长成看上去错误形状的硬件或模拟器怪癖。
- 刻意的偏离：为什么*没有*采用显而易见的做法，而代码本身会让读者想去"修"它。
- 能省去他人一轮调研的参考：规范章节、MSDN 结构体、已知 bug 链接。

绝不要写：

- 复述代码（`// increment the counter`、`// call the handler`、`// loop over the modules`）。
- 函数内部的小节标题或横幅（`// --- setup ---`、`// Step 3: cleanup`）。
- 对自己本次修改或推理的叙述（`// changed to fix X`、`// this is safe because ...`、`// now uses Y`）。
  这是在跟评审者说话，不是跟下一位读者，且合并那一刻起就是噪音。
- 只是用散文复述签名的 docstring 式头注释。
- 显然的类型或作用域信息（`// pointer to the process`）。

如果不确定某条注释是否够格，那就是不够。别写。零注释的改动是完全合格的改动；给本可读的代码撒注释的改动会被打回。

## 开发须知

- 对改动的文件运行 clang-format 以保证格式一致
- 不要走捷径或用变通方案。始终优先干净、地道的方案。
- 绝不提交二进制文件（可执行文件、DLL、目标文件、归档、图片、编译后的样本）。Git LFS 指针和生成产物不是变通办法。

## Pull Request

- 一个 PR 只包含一个关注点。无关的修复、重构或优化应放进各自的 PR。
- 标题与描述准确描述完整 diff。不要留下未记录的变更。
