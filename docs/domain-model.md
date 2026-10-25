# 领域模型

一页看懂核心对象与三条硬规则。记忆流水线、检索与可靠性细节见 [design-notes.md](design-notes.md)。

```mermaid
classDiagram
    class Pet {
        +str pet_id
        +str user_id
        +str kind
    }
    class Run {
        +str run_id
        +str client_request_id
        +RunStatus status
        +int generation
        +VersionTuple version
        +transition_to()
        +bump_generation()
    }
    class VersionTuple {
        +str graph
        +str prompt_bundle
        +str model_route
        +str index_version
    }
    class NodeEvent {
        +str run_id
        +int seq
        +str node
        +EventType type
    }
    class Proposal {
        +ProposalKind kind
        +str source_node
        +int generation
        +dict payload
    }
    class CommitPlan {
        +RunStatus final_status
        +str reply
        +list~Effect~ effects
    }
    class Effect {
        +EffectKind kind
        +str idempotency_key
    }
    class Episode {
        +str episode_id
        +int turn_start
        +int turn_end
    }
    class Fact {
        +str fact_id
        +MemoryViewpoint viewpoint
        +FactStatus status
        +tuple refutes
    }
    class Profile {
        +str dimension
        +int version
        +list~str~ fact_ids
    }

    Pet "1" --> "*" Episode : 记忆域
    Run --> Pet : 绑定当前宠物
    Run --> VersionTuple : 冻结版本
    Run "1" --> "*" NodeEvent : 事件流
    Proposal --> CommitPlan : settle()
    CommitPlan --> Effect : 副作用
    Episode --> Fact : 提炼
    Fact --> Profile : summarized_into
```

## 三条硬规则

1. **AI 只提议**：Proposal 不是结果，必须经 `settle()` 校验后才产生 CommitPlan。
2. **终态不可逆**：Run.status 到达 `succeeded / failed / cancelled` 后永不再变。
3. **副作用带幂等键**：每个 Effect 都有 `idempotency_key`，重试不会做两遍。
