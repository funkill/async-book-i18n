> [task-model/chapter.md](https://github.com/aturon/apr/blob/ffb00140a767d6e7a4a8875bf6965d10f830a271/src/task-model/chapter.md)
> commit ffb00140a767d6e7a4a8875bf6965d10f830a271

# 任务与执行器
为了用Rust高效地写异步代码, 你需要了解它的基础: 任务模型. 幸运的是, 该模块仅由
几个简单的模块组成.

本章将深层介绍任务概念, 然后阐释怎样通过构建能可工作的任务执行器和
事件循环, 以及整合它们来让系统运行起来.

