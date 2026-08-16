# MIMO学习与面试资料

本目录集中保存Codex基于F0工程整理的工程导读、收发端模块学习记录和数字前端面试资料。工程源码仍保留在上一级目录；每日聊天归档继续保存在`../docs/chats`。

## 推荐入口

1. [MIMO项目快速浏览与关键模块复现指南](./MIMO项目快速浏览与关键模块复现指南.md)：十分钟项目主线、全链路数据变化、信道估计/MMSE/Turbo单测顺序和高频面试题。
2. [工程导读](./工程导读.md)：第一次浏览整个F0工程时从这里开始。
3. [MIMO收发端总结与项目面试问答](./MIMO收发端总结与项目面试问答.md)：收发端MIMO主线、DRAM关联和项目深挖。
4. [紫光国芯数字前端面试准备](./紫光国芯数字前端面试准备.md)：自我介绍、前端基础与手撕RTL。

## 发射端学习

- [MIMO_TX_Top图解](./MIMO_TX_Top_图解.md)
- [MIMO_TX_Top子模块学习索引](./MIMO_TX_Top子模块学习索引.md)
- [TX_BIT_Processor学习](./TX_BIT_Processor学习.md)
- [TX_BIT_FIFO_Exchange学习](./TX_BIT_FIFO_Exchange学习.md)
- [TX_MIMO_Processor学习](./TX_MIMO_Processor学习.md)
- [TX_RRH_FIFO_Exchange学习](./TX_RRH_FIFO_Exchange学习.md)
- [TX_RRH_Processor学习](./TX_RRH_Processor学习.md)
- [Over_Sample_Group学习](./Over_Sample_Group学习.md)

## 接收端学习

- [MIMO_RX_Top子模块学习索引](./MIMO_RX_Top子模块学习索引.md)
- [RX_RRH_Processor学习](./RX_RRH_Processor学习.md)
- [RX_RRH_FIFO_Exchange学习](./RX_RRH_FIFO_Exchange学习.md)
- [RX_MIMO_Processor学习](./RX_MIMO_Processor学习.md)
- [RX_BIT_Processor学习](./RX_BIT_Processor学习.md)

## 文件管理约定

- 学习和面试Markdown统一保存在本目录。
- 工程RTL、IP核、PDF和原始资料不移动。
- 每日聊天记录仍写入`../docs/chats/YYYY-MM-DD-F0聊天记录.md`。
- 后续新增的MIMO学习文档也应放在本目录，并同步更新本索引。
