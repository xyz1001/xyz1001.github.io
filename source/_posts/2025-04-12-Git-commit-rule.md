---
title: Git提交信息规范和配套工具
author: 张帆
tags:
  - Git
date: 2025-04-12 11:21:22
---

## 说明

Git提交信息是维护者对一个代码仓库的历史修改极其重要的了解途径，提交信息不足会大大影响代码的可维护性。但在实际业务开发中，经常出现提交信息编写随意，难以约束的问题。为了保证提交信息的完整性和可用性，我在业务内部对提交格式做一些要求，并提供工具保证规范的落地。该规范更多的针对于C++应用开发场景，可能不适用于所有团队和场景，仅供参考。

<!--more-->

## 规范

结合内部业务的过往经验，以及[阿里](https://zhuanlan.zhihu.com/p/182553920)和 [Angular](https://github.com/angular/angular/blob/22b96b9/CONTRIBUTING.md#-commit-message-guidelines) 提交规范，制定以下提交规范：

1. 提交内容应当使用开发者最熟悉的语言。在开发者对多门语言都很熟悉时，优先选择团队成员都熟悉的语言。保证提交内容的准确性，完整性和可读性是首要目标。

2. 提交内容应当使用 UTF-8 without BOM 编码，避免在不同平台下的乱码问题。

3. 每次提交均需修改项目配置文件中的版本号，如 `conanfile.py` 或 `setup.py` 等。对于依赖库类型的仓库，版本应遵循语义化版本规则。

4. 提交格式应遵循以下标准：

```
[<type>][<version>][<jira>] <title>
why: xxx
how: xxx
influence: xxx
```

### 各字段说明

- **type**: 提交类型，为以下关键词中的一个，需使用方括号包围。提交类型应当根据本次提交的目的来确定，而不是修改内容。如若更新依赖库的目的是为了实现一个新功能，提交类型应当为 feature 而不是 chore。

  - `feature`：本次提交用于完成一个功能特性，该类型的提交通常应当填写 jira 字段，链接到一个需求类型的 JIRA
  - `bugfix`：本次提交用于修复一个 bug，该类型的提交通常应当填写 jira 字段，链接到一个缺陷类型的 JIRA。该类型提交必须填写 why/how/influence 字段
  - `refactor`：本次提交用于重构代码，提高代码的可维护性和可读性，对实际功能没有影响，也没有解决现有的 bug
  - `depend`：本次提交用于更新依赖
  - `build`：本次提交用于修改构建脚本，通常不会影响应用行为
  - `test`：本次提交用于修改测试代码，如 test 或 example 相关的代码
  - `perf`：本次提交用于提升性能和体验
  - `doc`：本次提交用于修改文档
  - `style`：本次提交用于调整代码格式
  - `revert`：本次提交用于回滚某些提交
  - `chore`：其他杂项，尽量不要使用

- **version**：版本号，为 x.x.x 格式，如 1.0.0，需使用方括号包围

- **jira**：jira 号，格式为 XXX-xxx，前半部分为大写英文字符，后半部分为数字，需使用方括号包围。可省略，但强烈要求在有对应 jira 的情况下填写该项，如使用其他问题跟踪平台则修改为其他平台问题ID

- **title**：简要描述

- **why**：本次提交的问题原因，在提交类型为 bugfix 时该字段必填

- **how**：本次提交的问题解决方式，在提交类型为 bugfix 时该字段必填

- **influence**：本次提交的影响范围，在提交类型为 bugfix 时该字段必填

### 示例

```
# Bug修复示例
[bugfix][4.0.2][SEEWOOS-155622] 修复磁盘不足时崩溃的问题
why: 磁盘不足时创建文件失败导致崩溃
how: 添加对剩余磁盘空间的检测
influence: xx功能

# 新功能示例
[feature][1.2.0][SEEWOOS-155623] 添加自动重试机制
why: 网络不稳定时容易失败
how: 添加最多3次重试逻辑

# 依赖更新示例
[depend][1.2.1] 更新 boost 到 1.81.0
why: 修复boost中的安全漏洞

# 文档更新示例
[doc][1.2.1] 补充 README 中的编译说明

# 重构示例
[refactor][1.2.2] 重构网络连接模块
why: 提高代码可维护性
how: 抽象出连接管理类
```

## 工具

为了保证以上规范的持续执行和简化操作，我们提供了两个工具。

### [imit](https://github.com/leytou/imit)

#### 简介

该工具用于代替 `git commit` 命令，在提交时通过交互式对话的方式填写上述字段。jira 数据会从 jira 网站上获取当前用户下的 jira 列表并显示，该工具目前仅支持jira作为问题跟踪平台数据来源，若有需要支持其他平台，欢迎提交PR。

#### 使用文档

1. 通过 pip 安装/更新：
```bash
pip install imit -U
```

2. 首次安装后，需执行 `imit config` 配置 jira 信息，包括：
   - jira 网址：如https://jira.xxx.com/
   - API token：jira API Token，可在jira主页面右上角-用户信息-个人访问令牌中创建

3. 在需要执行 `git commit` 的地方执行 `imit`

#### 使用效果

![imit使用效果](imit.png)

### [pre-commit-hook-cpp](https://github.com/xyz1001/pre-commit-hooks-cpp)

#### 简介

该工具基于 pre-commit 框架，提供了对 C++ 项目的 git hook 支持。当前仅支持 check-commit-msg 一个 hook，可用于检测提交信息的字符编码以及格式。用户可通过提供校验规则正则来自定义提交信息检查规则。如满足以上要求的正则如下

``` regex
(?:^(?:fixup! )*\[(?:(?:feature)|(?:chore)|(?:refactor)|(?:revert)|(?:perf)|(?:test)|(?:doc)|(?:style)|(?:improve)|(?:build)|(?:depend)|)\]\[\d+\.\d+\.\d+\](?:\[[A-Z]+\-\d+\])? .+(?:\n^why: .*)?(?:\n^how: .*)?(?:\n^influence: .*)?$)|(?:^(?:fixup! )*\[bugfix\]\[\d+\.\d+\.\d+\](?:\[[A-Z]+\-\d+\])? .+\n^why: .+\n^how: .+\n^influence: .+$)
```

#### 使用文档

1. 通过 pip 安装 pre-commit:
```bash
pip install pre-commit -U
```

2. 在项目根目录下创建 `.pre-commit-config.yaml` 文件，并参考项目 README 中说明填写内容

3. 执行以下命令：
```bash
git add .
pre-commit install
```

为了确保使用者不会因为遗忘安装pre-commit，我们可以在项目构建脚本中添加自动安装逻辑，如对于CMake，我们可以添加如下代码到 CMakeList.txt 中

``` cmake
execute_process(COMMAND pre-commit install RESULT_VARIABLE result)
if(NOT result EQUAL 0)
    message(FATAL_ERROR "pre-commit is not installed. Please install it with 'pip install pre-commit'.")
endif()
```
#### 使用效果

![pre-commit-hook-cpp使用效果](pre-commit-hook-cpp.png)
