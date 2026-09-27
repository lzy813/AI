# 一、简介

## 1、定位

- LangGraph 被称为<font color="red">**AI Agent 的操作系统**</font>，它以可靠的持久化执行和完善的人机协同能力，在 AI Agent 开发中占据核心地位。当前已全面进入 AI Agent 时代，各大公司都在构建独特的 Agent 产品，LangGraph 是构建复杂 Agent 系统的首选框架

- LangGraph 与 LangChain 的关系

  - 两者并非对立，而是<font color="red">**协同工作**</font>的关系：

  | 框架      | 定位             | 适用场景                         |
  | --------- | ---------------- | -------------------------------- |
  | LangChain | 顶层应用层设计   | 快速上手、简单直观的 Agent 构建  |
  | LangGraph | 底层源码架构编排 | 复杂逻辑、持久化检查点、人机协同 |

- LangGraph 与 LangChain 的对比

  | 对比维度 | LangChain             | LangGraph                      |
  | -------- | --------------------- | ------------------------------ |
  | 定位     | Agent 高层开发框架    | 底层编排框架 & Agent Runtime   |
  | 核心入口 | create_agent          | StateGraph / @entrypoint       |
  | 适用场景 | 结构直接的 Agent 应用 | 复杂工作流、持久化、长时间运行 |
  | 流程控制 | Agent 循环自动管理    | 精细控制节点、边、条件分支     |
  | 学习成本 | 较低                  | 较高（但能力上限更高）         |