# Phase Definitions and Guardrails

## Phase 0 (Frozen Baseline)
- **Definition**: Architecture and engineering baseline is frozen; used only as stable starting point.
- **Tag**: `phase0` (read-only, must never move).
- **Entry**: Baseline established and buildable; no further changes allowed.
- **Exit / Acceptance**: Tag `phase0` exists; build passes; no pending work.
- **Allowed**: Checkout `phase0` for inspection only.
- **Prohibited**: Modifying any Swift/Metal/Xcode engineering code; git reset/rebase to `phase0`; force push involving `phase0`.
- **Rollback Philosophy**: Revert-only. Any rollback must use `git revert`; history on `main` only appends forward.

## Phase 0.5 (Guardrails / Versioning)
- **Definition**: Establishes “revertable, history-safe” development guardrails; no feature work.
- **Entry**: Phase 0 complete; working from `main` or a branch derived from `main`.
- **Exit / Acceptance**: Guardrail docs and policies in place; Phase 0 tag respected; revert-only rule documented; no code changes to the baseline.
- **Allowed**: Add or update documentation under `docs`.
- **Prohibited**: Modifying any Swift/Metal/Xcode engineering files.
- **Rollback Philosophy**: Revert-only. Use `git revert` for any backward change; do not rewrite history.

## Phase 1 | One-shot 3D Build (MVP)

### Phase 1 总定义

- **名称**: Phase 1 | One-shot 3D Build (MVP)
- **插件策略**: Plugin B: On-device Fast Fit（端上快速拟合式，非训练式）
- **核心特性**: 一次性资产构建，用户只能重来，不能补拍增量修补（增量补拍放到未来 Phase）

### Phase 1 三条硬成功标准

1. **T+1–2s 必须显示可漫游结果**（哪怕粗）
2. **不因机型差而黑屏/卡死**：宁可降级
3. **统一资产格式 + progressive 渲染管线跑通**（为后续插件铺路）

### Phase 1 进入与退出

- **Entry**: Must branch from `phase0.5` (or `main` after it contains Phase 0.5 guardrails).
- **Allowed**: Feature implementation on branches derived from `phase0.5`.
- **Prohibited**: Direct push to `main`; force push; reset/rebase to `phase0`; breaking the revert-only rule.
- **Rollback Philosophy**: Any breaking change is undone via `git revert`; `main` history must only grow forward.

---

### Phase 1-0: 接口合同冻结（Router/Plugin Contract）

#### Goal
冻结 Router 与 Plugin 之间的接口与边界，确保团队协作有明确的"合同"。

#### Scope
- **做什么**:
  - 定义 `RouterPlan` 结构（预算参数、停止规则、progressive 输出）
  - 定义 `Budget` 接口（time_budget_ms / frame_budget / max_splats / lod / sh_order）
  - 定义 `StopRules` 接口（热/电/内存触发条件）
  - 定义 `Progressive` 输出格式（渐进式质量提升的数据结构）
- **不做什么**:
  - 不实现 Router 或 Plugin 的具体算法
  - 不涉及设备检测或性能测量

#### Deliverables
- **代码**: 接口定义文件（structs/protocols，占位实现）
- **文档**: 至少 2 个文档：
  - `docs/ROUTER.md`: Router 接口规范、输入输出格式
  - `docs/PLUGIN.md`: Plugin 接口规范、调用约定
- **格式**: 接口定义必须可编译，但实现可以为空

#### Entry Criteria
- Phase 0.5 完成
- 团队对齐 Plugin B 策略（端上快速拟合）

#### Exit Criteria / Acceptance
- ✅ `RouterPlan` / `Budget` / `StopRules` / `Progressive` 接口定义完成
- ✅ 至少 2 个接口文档（ROUTER/PLUGIN）已写入 `docs/`
- ✅ 接口定义可编译（无语法错误）
- ✅ 团队 review 通过接口设计

#### Rollback Note
如需回滚到 Phase 1-0 之前：
```bash
git checkout -b rollback/before-phase1-0 main
git revert <phase1-0-commit-hash>
# 创建 PR 并合并
```

---

### Phase 1-1: Router v0（设备分层 + 预算输出 + 安全刹车）

#### Goal
实现 Router v0，能够根据设备能力和素材特征输出预算参数，并具备安全刹车机制。

#### Scope
- **做什么**:
  - 实现 `DeviceTier` 分层（L/M/H，按能力不按机型列表）
  - 实现设备状态检测（热/电/内存）
  - 实现素材特征分析（N、coverage、quality、parallax 最小集）
  - 实现预算计算（time_budget_ms / frame_budget / max_splats / lod / sh_order / progressive / stop_rules）
  - 实现 stop_rules 触发机制（至少日志可见）
- **不做什么**:
  - 不实现 Plugin B 的具体拟合算法
  - 不实现渲染管线

#### Deliverables
- **代码**: 
  - `Core/Pipeline/Router/DeviceTier.swift`: 设备分层实现
  - `Core/Pipeline/Router/RouterPlan.swift`: 预算计算实现
  - `Core/Pipeline/Router/StopRules.swift`: 安全刹车实现
- **文档**: Router v0 使用说明
- **日志**: stop_rules 触发时的日志输出

#### Entry Criteria
- Phase 1-0 完成（接口合同已冻结）
- 接口定义可编译

#### Exit Criteria / Acceptance
- ✅ 任何有效输入（设备状态 + 素材特征）必产出 `RouterPlan`
- ✅ 热/电/内存触发 stop_rules 时至少日志可见
- ✅ DeviceTier L/M/H 分层逻辑正确（可通过测试验证）
- ✅ 预算参数计算符合接口规范
- ✅ 单元测试覆盖主要路径

#### Rollback Note
如需回滚到 Phase 1-1 之前：
```bash
git checkout -b rollback/before-phase1-1 main
git revert <phase1-1-commit-hash>
# 创建 PR 并合并
```

---

### Phase 1-2: 最小链路跑通（只接 Plugin B，先出 Enter 结果）

#### Goal
实现最小链路，确保 T+1–2s 内能进入 Viewer 并显示可漫游结果。

#### Scope
- **做什么**:
  - 实现 Plugin B 最小拟合算法（端上快速拟合）
  - 实现 Router → Plugin B → Viewer 的完整链路
  - 实现 progressive 渲染（后续逐步变好，不要求极致质量）
  - 实现 cancel 机制
  - 实现 fail-soft：失败时返回可用结果或明确提示
- **不做什么**:
  - 不追求极致质量（质量一般即可）
  - 不实现 publish 档（放到 Phase 1-3）

#### Deliverables
- **代码**:
  - `Core/Training/PluginB/`: Plugin B 最小实现
  - `Core/Pipeline/Training/TrainingSession.swift`: 训练会话实现
  - `Features/Viewer/ViewerView.swift`: Viewer 基础实现
- **文档**: 最小链路使用说明
- **日志**: 链路各阶段的日志输出

#### Entry Criteria
- Phase 1-1 完成（Router v0 可用）
- Router 能输出有效预算参数

#### Exit Criteria / Acceptance
- ✅ **硬指标**: T+1–2s 内必须显示可漫游结果（哪怕粗）
- ✅ progressive 渲染工作正常（质量逐步提升可见）
- ✅ cancel 机制可用（用户可中断）
- ✅ fail-soft 机制可用（失败不崩，返回可用结果或明确提示）
- ✅ 能在 Viewer 中漫游（基础交互正常）

#### Rollback Note
如需回滚到 Phase 1-2 之前：
```bash
git checkout -b rollback/before-phase1-2 main
git revert <phase1-2-commit-hash>
# 创建 PR 并合并
```

---

### Phase 1-3: 可发布质量（Publish 档）+ 本地保存/加载闭环

#### Goal
实现 publish 档（10–30s 范围按 tier），并完成本地资产保存/加载闭环。

#### Scope
- **做什么**:
  - 扩展 `RouterPlan` 支持 publish 预算档（10–30s 范围按 DeviceTier）
  - 实现本地资产格式（先内部格式可）
  - 实现 save 功能（保存资产到本地）
  - 实现 load/reopen 功能（从本地加载资产）
- **不做什么**:
  - 不实现云同步（放到未来 Phase）
  - 不实现资产格式的向后兼容（先内部格式）

#### Deliverables
- **代码**:
  - `Core/Pipeline/Router/RouterPlan.swift`: publish 预算档支持
  - `Shared/Persistence/AssetFormat.swift`: 资产格式定义
  - `Shared/Persistence/AssetStorage.swift`: 保存/加载实现
  - `Features/Viewer/AssetLoader.swift`: 资产加载器
- **文档**: 资产格式规范、保存/加载使用说明
- **格式**: 本地资产文件格式（内部格式）

#### Entry Criteria
- Phase 1-2 完成（最小链路跑通）
- Enter 结果能正常显示

#### Exit Criteria / Acceptance
- ✅ RouterPlan 支持 publish 预算档（10–30s 范围按 tier）
- ✅ 能保存资产到本地（内部格式）
- ✅ 能从本地加载资产并显示
- ✅ reopen 功能正常（应用重启后能加载之前保存的资产）
- ✅ 资产格式定义清晰（可扩展）

#### Rollback Note
如需回滚到 Phase 1-3 之前：
```bash
git checkout -b rollback/before-phase1-3 main
git revert <phase1-3-commit-hash>
# 创建 PR 并合并
```

---

### Phase 1-4: Fail-soft + 稳定性（输入差也不崩）

#### Goal
确保输入差（垃圾素材）时也不崩不黑屏，能降级处理并返回首页继续。

#### Scope
- **做什么**:
  - 实现错误分类（输入错误、处理错误、设备错误等）
  - 实现用户可执行提示（明确告诉用户如何修复）
  - 实现降级策略：
    - 减少帧率
    - 降低 splats 数量
    - 降低 sh_order
    - 提前 stop
  - 实现错误恢复机制（失败后能返回首页继续）
- **不做什么**:
  - 不追求完美处理所有边界情况（先覆盖主要错误类型）
  - 不实现自动修复（只提示用户）

#### Deliverables
- **代码**:
  - `Shared/Errors/ErrorTypes.swift`: 错误分类定义
  - `Core/Pipeline/Router/DegradationStrategy.swift`: 降级策略实现
  - `Features/Capture/ErrorHandling.swift`: 错误处理 UI
- **文档**: 错误处理规范、降级策略说明
- **日志**: 错误分类和降级触发的日志

#### Entry Criteria
- Phase 1-3 完成（publish 档和保存/加载可用）
- 基础链路稳定

#### Exit Criteria / Acceptance
- ✅ 喂垃圾素材不崩不黑屏（至少返回错误提示）
- ✅ 错误分类清晰（至少覆盖主要错误类型）
- ✅ 用户可执行提示明确（用户知道如何修复）
- ✅ 降级策略可用（减少帧/降低 splats/降低 sh_order/提前 stop）
- ✅ 失败后能返回首页继续（不卡死）

#### Rollback Note
如需回滚到 Phase 1-4 之前：
```bash
git checkout -b rollback/before-phase1-4 main
git revert <phase1-4-commit-hash>
# 创建 PR 并合并
```

---

### Phase 1-5: Metrics & Replay（为 Router v1 调参做准备）

#### Goal
记录关键指标，为 Router v1 调参做准备，支持本地 session 回放。

#### Scope
- **做什么**:
  - 实现指标记录：
    - `tt_preview` (time to preview)
    - `tt_publish` (time to publish)
    - `stop_reason` (停止原因)
    - `frames_used` (使用的帧数)
    - `tier` (设备层级)
    - `budgets` (预算参数)
    - `crash-free session` (无崩溃会话)
  - 实现本地落盘 `session.json`
  - 实现 session 回放（可选，基础即可）
- **不做什么**:
  - 不实现云上传（放到未来 Phase）
  - 不实现复杂的分析工具（先记录数据）

#### Deliverables
- **代码**:
  - `Shared/Infrastructure/Metrics.swift`: 指标记录实现
  - `Shared/Persistence/SessionStorage.swift`: session 存储实现
  - `Core/Pipeline/MetricsCollector.swift`: 指标收集器
- **文档**: 指标定义文档、session 格式说明
- **格式**: `session.json` 格式定义

#### Entry Criteria
- Phase 1-4 完成（fail-soft 和稳定性可用）
- 主要功能稳定

#### Exit Criteria / Acceptance
- ✅ 所有关键指标都能记录（tt_preview / tt_publish / stop_reason / frames_used / tier / budgets / crash-free session）
- ✅ 本地落盘 `session.json` 正常（格式正确）
- ✅ session 数据可用于分析（至少能读取和解析）
- ✅ 指标记录不影响性能（异步记录，不阻塞主流程）

#### Rollback Note
如需回滚到 Phase 1-5 之前：
```bash
git checkout -b rollback/before-phase1-5 main
git revert <phase1-5-commit-hash>
# 创建 PR 并合并
```

---

### Phase 1 与 Phase 2 的边界

**Phase 1 完成标准**:
- Phase 1-0 到 1-5 全部完成
- 三条硬成功标准全部满足
- 所有子阶段的 Exit Criteria 通过

**Phase 2 才引入的内容**:
- **训练式精修插件 A**（Plugin A: Training-based Refinement）
- **限定条件**: 仅 DeviceTier H 且需热/电允许
- **增量补拍**: 支持用户补拍增量素材进行精修
- **云训练**: 可选云训练支持（如果设备能力不足）

**Phase 1 不包含**:
- ❌ 训练式插件（Plugin A）
- ❌ 增量补拍功能
- ❌ 云训练支持
- ❌ 极致质量追求（质量一般即可，满足 T+1–2s 进入）

**回滚到 Phase 1 之前**:
如需回滚整个 Phase 1：
```bash
git checkout -b rollback/before-phase1 main
git revert <phase1-start-commit>..<phase1-end-commit>
# 创建 PR 并合并
```






