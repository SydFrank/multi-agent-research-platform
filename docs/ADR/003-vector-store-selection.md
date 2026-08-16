# ADR 003 · 向量存储选型:pgvector + VectorStore 抽象层

- **状态**:已采纳
- **相关**:[ARCHITECTURE.md §8.6](../ARCHITECTURE.md#86-vectorstore-抽象层)

---

## 背景

RAG 子系统需要向量存储支持文档片段的嵌入检索。候选方案包括 pgvector、Milvus、Qdrant、FAISS、Chroma 等。

系统的检索有一个**硬性约束**:所有检索必须满足 Point-in-Time 语义,即只能返回 `published_at <= as_of_date` 的文档片段(见 [ADR 005](./005-point-in-time-semantics.md) 与 ARCHITECTURE §9)。这不是可选的业务过滤,而是保证回测结论有效性的正确性要求。

## 问题

选择哪种向量存储?如何在满足当前需求的同时,保留未来替换的可能?

## 决策

**默认实现采用 pgvector,并在 `rag` 模块中定义 `VectorStore` 抽象接口,使实现可替换。**

```python
class VectorStore(Protocol):
    async def upsert(self, items: list[ChunkVector]) -> None: ...
    async def search(
        self, embedding: list[float], top_k: int,
        filters: MetadataFilter,     # symbol / published_at / doc_type
    ) -> list[ScoredChunk]: ...
    async def delete(self, ids: list[str]) -> None: ...
```

## 理由

### 决定性因素:元数据过滤与向量检索的耦合

本系统的检索**强依赖元数据过滤**,且过滤条件具有正确性含义。这带来一个关键的架构差异:

| 方案 | 过滤与检索的关系 | 风险 |
|---|---|---|
| **pgvector** | 在同一条 SQL 中完成:`WHERE published_at <= :as_of AND symbol = ANY(:symbols) ORDER BY embedding <=> :q LIMIT k` | 无 |
| 独立向量库 | 需将文档元数据同步一份到向量库;或先向量召回再回业务库过滤 | ① 元数据同步的一致性风险 ② **先召回后过滤会导致召回不足**——若 top-k 结果大量被时间过滤掉,实际可用片段远少于 k |

第二点尤其关键:在投研场景中,同一标的的文档在时间上高度密集,`as_of_date` 过滤可能筛掉大部分近期文档。"先召回 50 条再过滤"很可能只剩个位数,而 pgvector 的过滤发生在检索内部,返回的始终是满足约束的 top-k。

### 次要因素

- **运维成本**:业务数据、向量、LangGraph 检查点同在一个 Postgres 实例,减少组件数量与备份/迁移复杂度
- **事务一致性**:文档摄取与向量写入可在同一事务中提交,不存在"文档已入库但向量未写入"的中间态
- **规模匹配**:当前预期向量规模在百万级以内,HNSW 索引(`m=16, ef_construction=64`)性能充足

### 为什么仍要抽象层

- 规模增长后可能需要迁移,抽象层将改动局限于单个实现类
- 评测体系可对不同实现做同口径对比,使迁移决策有数据依据
- 抽象接口本身强制了"过滤必须作为检索参数传入"的设计,任何实现都不能绕过 PIT 约束

## 备选方案评估

| 方案 | 优势 | 劣势 | 结论 |
|---|---|---|---|
| **pgvector** | 过滤与检索同查询;运维简单;事务一致 | 超大规模性能弱于专用库;索引类型少 | ✅ 采纳 |
| Milvus / Qdrant | 大规模性能强;分布式;索引类型丰富 | 需独立部署;元数据需同步;存在召回不足问题 | 规模增长后再评估 |
| FAISS | 单机性能高;无服务依赖 | 无持久化、无元数据过滤,需自行封装 | 仅适合离线实验 |
| Chroma | 上手快 | 生产特性与规模能力有限 | 不适合 |

## 后果

**正面**
- 检索的 PIT 正确性由数据库层保证,不依赖应用层拼接顺序
- 部署组件减少,`docker compose` 启动更快
- 摄取管线的一致性语义简单

**负面**
- 向量规模增长时,Postgres 实例会同时承载业务查询与向量检索的负载
- HNSW 索引构建在大批量摄取时会成为瓶颈

**缓解**
- 监控向量检索延迟(`retrieval_latency_seconds{stage="vector"}`),接近阈值时评估迁移
- 大批量摄取采用离线构建索引的方式,避免影响在线查询

## 迁移触发条件

满足以下任一条件时,启动向量库迁移评估:

1. 向量规模超过千万级
2. 向量检索 p95 延迟持续超过阈值,且索引调优无法解决
3. 需要 Postgres 不支持的索引类型或检索模式
4. 向量检索负载与业务查询产生明显资源竞争,需要独立扩容

迁移时需通过评测体系验证:新实现在同一评测集上的 Recall/MRR/NDCG 不低于现有实现。
