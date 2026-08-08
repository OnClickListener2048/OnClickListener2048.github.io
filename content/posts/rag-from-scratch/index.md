---
title: "从零手写一个 RAG 系统（实战学习笔记）"
date: 2026-08-09T00:00:00+08:00
draft: false
weight: 1
tags: ["AI", "LLM", "RAG", "embedding", "向量检索", "Ollama", "bge-m3", "Python"]
categories: ["人工智能", "学习笔记"]
author: "OnClickListener"
description: "不用任何 RAG 框架，从 embedding 到分块、向量存储、检索全链路手写一遍：记录相似度矩阵里的意外发现、老显卡上的 Ollama 排坑，以及一次真实的检索质量验证。"
---

去年写过一篇 [RAG 概念科普](/posts/rag-concept/)，这次不一样：**在本地从零手写了一个最小 RAG 系统**。不引入 LlamaIndex、LangChain 的检索部分，链路里的每一环——分块、embedding 调用、向量存取、相似度检索——都是自己写的。

这么做的原因很简单：目标是**学习**，而只有手写一遍，才能回答那些框架文档不会告诉你的问题：

- embedding 到底返回了什么？数值长什么样？
- 为什么"苹果"和"汽车"的相似度也有 0.53？
- 检索为什么不能只看 top-k，还要看分数？
- 全量遍历检索在什么时候会撑不住？

这篇文章是完整的过程记录，包含所有实验结果和踩坑经历。

## 一、环境：老显卡上的 Ollama

选型结论：本地模型 `bge-m3`（1024 维，中文效果好，离线免费），通过 Ollama 提供 OpenAI 兼容的 `/v1/embeddings` 接口。

踩了个大坑：本机是 GTX 1060（Pascal 架构，太老），新版 Ollama 的 CUDA 内核一加载就崩：

```
llama-server process has terminated:
CUDA error: the provided PTX was compiled with an unsupported toolchain
```

排查过程值得记录（每个变量都试过）：

| 尝试 | 结果 |
|---|---|
| `OLLAMA_GPU_LAYERS=0`（0 层上 GPU） | ❌ 无效，CUDA 仍初始化 |
| `OLLAMA_VISIBLE_DEVICES=""`（屏蔽 GPU） | ❌ 无效，空值被当成"全部可见" |
| `OLLAMA_LLM_LIBRARY=cpu`（强制 CPU 推理库） | ✅ 生效 |

经验：**embedding 是纯推理（无生成），小模型跑 CPU 完全够用**。12 个分块建索引总共才 2 秒。以后遇到老显卡 + 新 CUDA 内核的组合，`OLLAMA_LLM_LIBRARY=cpu` 是第一选择。

## 二、阶段 0：embedding 到底长什么样

先不急着写 RAG，先搞懂向量本身。写了个探索脚本：调 `/embeddings` 拿向量，手写余弦相似度（`dot / (|a|·|b|)`），算相似度矩阵。

### 向量性质

```
向量维度: 1024
「苹果」前 10 个元素: [-0.027, 0.006, -0.074, ...]
各向量 L2 范数: [1.0, 1.0, 1.0, ...]   <- API 已归一化
```

### 相似度矩阵：两个意外发现

| 词对 | 相似度 | 解读 |
|---|---|---|
| 苹果 ↔ 苹果公司 | **0.91** | 一词多义被捕捉（同一词的不同语境） |
| 计算机 ↔ 电脑 | **0.85** | 同义词 |
| 香蕉 ↔ 水果 | 0.71 | 同类概念 |
| 苹果 ↔ 汽车 | 0.53 | ⚠️ 无关词竟然也有 0.53 |

**发现一**：embedding 能区分一词多义，"苹果"这个向量同时靠近"水果"（0.68）和"苹果公司"（0.91）——一个向量承载了两种语义。

**发现二（重要）**：词级相似度整体基线偏高，无关词也有 0.5+。这是 BGE 类模型对**短文本的各向异性**——embedding 偏向于少数几个方向。**含义：孤词检索不可靠，RAG 必须用段落级文本做分块**。这个发现直接决定了后面分块器的设计。

### 句级区分度：真正可用的信号

```
「删除对话记录」vs「清空聊天历史」→ 0.82   （同义句）
「删除对话记录」vs「写秋天的诗」  → 0.38   （无关句）
```

句子层面区分度清晰，这才是检索能用的区间。

### 一个数学验证

normalize 之后 `cos(a,b) == dot(a,b)`（实测 0.8207 == 0.8207）。这就是为什么向量库都存归一化向量、用点积加速——**余弦相似度只关心方向，不关心模长**。

## 三、阶段 1：手写最小 RAG 链路

链路四步：**分块 → embedding → 存 SQLite → 手写检索**。

### 分块器：结构感知

不用固定长度硬切，而是按 Markdown 标题切分，维护"标题路径"作为语义上下文前缀：

```python
# 核心思路（简化）
for line in md_text.splitlines():
    if m := HEADING_RE.match(line):      # 遇到标题
        flush_section()                  # 保存上一个 section
        heading_stack = [h for h in heading_stack if h[0] < m.level]
        heading_stack.append((m.level, m.title))   # 维护标题栈
    else:
        current_body.append(line)

# 标题路径拼进 chunk 内容，让 embedding 知道这段属于哪个主题
content = f"{' > '.join(path)}\n{body}"
```

超长 section 按句子边界滑窗 + 重叠（防止句子跨块被切断），过短的碎块与相邻块合并。

### 存储：向量进 SQLite BLOB

不急着上向量库——先用手写最朴素的方式：`struct` 把 1024 个 float32 打包成 BLOB 存普通表，每行 4096 字节：

```python
def pack_embedding(vec): return struct.pack(f"{len(vec)}f", *vec)
def unpack_embedding(blob): return list(struct.unpack(f"{len(blob)//4}f", blob))
```

### 检索：全量遍历 + 手写余弦

```python
for row_id, path, content, blob in rows:
    vec = unpack_embedding(blob)
    scored.append((cosine_similarity(query_vec, vec), path, content))
scored.sort(key=lambda s: s[0], reverse=True)
```

O(n) 全量遍历——12 个 chunk 时毫秒级，毫无压力。**刻意不优化，留到下一阶段对比。**

## 四、验证：5/5 命中 + 一个阈值发现

用一篇分 12 块的中文演示文档（内容涵盖依赖注入、同步协议、记忆系统、工具系统等）跑查询：

| 查询 | top-1 命中 | 分数 |
|---|---|---|
| Koin 怎么做依赖注入？ | 依赖注入（Koin）✅ | **0.77** |
| 数据同步怎么工作？ | 同步协议 ✅ | **0.75** |
| 记忆怎么从聊天提取？ | 记忆系统·记忆提取 ✅ | 0.66 |
| 有哪些内置工具？ | 工具系统·内置工具 ✅ | **0.74** |
| 窄屏下界面怎么布局？ | 界面与响应式布局 ✅ | **0.70** |

**最有价值的发现来自一个负面用例**——查询"如何配置 CI 流水线？"（文档里没有的内容）：

```
top-1 分数: 0.437，命中的还是无关 chunk
```

对比正面查询 0.66~0.77 的分数，存在一个**无相关内容时的相似度基线（约 0.42~0.45）**。工程含义很直接：

> 真实系统不能只看 top-k，必须设阈值（如 0.55）。低于阈值就不把结果注入 prompt，否则模型会被无关上下文污染，甚至一本正经地编造答案。

这个 0.42 基线也提醒我：embedding 分数是**相对**信号，不同模型、不同文本长度的基线完全不同，阈值要靠实测，没有万能值。

## 五、性能基线（留给下一阶段对比）

- 建索引：12 chunk，CPU embedding，约 2 秒
- 查询：约 0.7 秒（其中查询向量 embedding 占 0.6 秒，检索本身毫秒级）

## 总结与下一步

这一轮学到的东西，比用任何框架都多：

1. **embedding 是方向不是坐标**——归一化后只用点积，这是所有向量库优化的起点
2. **短文本各向异性**决定了分块粒度必须是段落级
3. **检索质量 = 分块质量 × 模型质量 × 阈值设置**，三者都要实测
4. **朴素实现是最好的教科书**——先 O(n)，才能理解为什么需要索引

**下一步（阶段 2）三个对比实验**：

1. 手写全量遍历 vs `sqlite-vec`：把数据放大到几千 chunk，亲眼看性能退化
2. 手写 BM25 vs 语义检索：关键词型查询谁赢
3. 分块策略 A/B：固定长度 vs 结构感知

全部代码在 `fat-ai-server/scripts/`（`explore_embedding.py`、`chunker.py`、`rag_demo.py`），完全可复现。
