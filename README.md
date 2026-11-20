# 图片转3D珠宝定制系统

> 架构设计文档

---

## 1. 文档说明

### 1.1 目的

本文档用于指导「图片转3D珠宝定制系统」的技术选型、系统拆分、模块设计与后续开发落地，作为研发、测试、运维的统一参考依据。

### 1.2 范围

覆盖从用户上传单张珠宝图片，到生成可交互3D模型、参数调节、虚拟试戴、结果导出的完整链路。

### 1.3 术语

| 术语       | 说明                                    |
| ---------- | --------------------------------------- |
| GLB        | glTF 2.0 二进制格式，最终交付与展示格式 |
| 参数化模板 | 预设的戒托/宝石可调参数模型             |
| 粗模       | AI直接生成的低精度网格                  |
| 精模       | 经过对齐、修复、材质处理后的可用模型    |
| Task       | 一次完整的图片转3D任务                  |

---

## 2. 项目概述

### 2.1 背景

传统珠宝定制依赖人工建模，周期长、成本高。本系统希望通过「单图输入 + AI重建 + 参数化约束」的方式，快速生成可交互、可调节、可导出的3D珠宝模型，降低定制门槛。

### 2.2 核心价值

- 用户仅需一张照片即可获得3D预览
- 支持实时换金属、换宝石、调节尺寸
- 支持虚拟试戴
- 结果可导出给工厂或设计师继续加工

### 2.3 设计原则

1. **混合策略**：AI重建负责快速出形，参数化模板负责结构可控
2. **异步优先**：所有重计算任务异步化，保证前端响应速度
3. **可替换**：3D生成模型、识别模型均可插拔
4. **渐进增强**：先跑通MVP，再逐步提升精度与可控性

---

## 3. 需求概述

### 3.1 功能需求

| 编号 | 需求                                   | 优先级 |
| ---- | -------------------------------------- | ------ |
| F1   | 支持上传单张戒指/珠宝图片              | P0     |
| F2   | 自动去背景与图像规范化                 | P0     |
| F3   | 识别款式、主石形状、金属颜色、大致克拉 | P0     |
| F4   | 生成可交互3D模型（GLB）                | P0     |
| F5   | 支持旋转、缩放、灯光切换               | P0     |
| F6   | 支持更换金属材质与宝石颜色             | P1     |
| F7   | 支持调节主石大小、戒臂宽度等参数       | P1     |
| F8   | 虚拟试戴（摄像头手部追踪）             | P1     |
| F9   | 导出GLB/OBJ与分享链接                  | P1     |
| F10  | 任务进度实时反馈                       | P0     |

### 3.2 非功能需求

| 类型   | 指标                                              |
| ------ | ------------------------------------------------- |
| 性能   | 粗模生成 ≤ 15s（GPU），精模 ≤ 60s；查看器 ≥ 30FPS |
| 可用性 | 主流程操作步骤 ≤ 3步；支持PC与移动端              |
| 可靠性 | 任务失败可重试；关键服务具备超时与降级策略        |
| 扩展性 | 支持后续接入多图、视频、工厂API                   |
| 安全   | 上传文件类型与大小限制；原图与结果默认私有        |

---

## 4. 总体架构

### 4.1 架构全景

```text
┌─────────────────────────────────────────────────────────────────┐
│                        用户端（Web / H5）                         │
│  上传 | 进度 | Three.js查看器 | 参数面板 | 试戴 | 导出             │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API Gateway / 负载均衡                        │
│              鉴权 · 限流 · 文件校验 · 路由转发                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Upload Service │  │ Task Orchestrator│  │  Query Service  │
│  文件接收与存储  │  │  任务状态机调度  │  │  状态与结果查询  │
└────────┬────────┘  └────────┬────────┘  └─────────────────┘
         │                    │
         │                    ▼
         │         ┌──────────────────────┐
         │         │     消息队列          │
         │         │  (Redis Stream/Kafka) │
         │         └──────────┬───────────┘
         │                    │
         │         ┌──────────┼──────────┬──────────────┐
         ▼         ▼          ▼          ▼              ▼
┌─────────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ ┌────────────┐
│Image Process│ │Understand│ │3D Generate│ │Parametric  │ │Post-process│
│去背景/增强   │ │款式识别  │ │单图转3D   │ │模板对齐    │ │网格/材质   │
└─────────────┘ └──────────┘ └──────────┘ └────────────┘ └────────────┘
         │              │           │            │              │
         └──────────────┴───────────┴────────────┴──────────────┘
                                    │
                                    ▼
                     ┌──────────────────────────┐
                     │  对象存储 + PostgreSQL    │
                     │  + Redis + CDN            │
                     └──────────────────────────┘
```

### 4.2 分层说明

| 层级       | 职责                             |
| ---------- | -------------------------------- |
| 表现层     | 用户交互、3D渲染、试戴、参数调节 |
| 接入层     | 统一入口、安全防护、流量控制     |
| 业务服务层 | 任务管理、流程编排、业务逻辑     |
| AI计算层   | 图像理解、3D生成、网格后处理     |
| 数据层     | 持久化存储、缓存、文件管理       |
| 基础设施层 | 容器、GPU节点、监控、日志        |

---

## 5. 核心模块详细设计

### 5.1 前端模块

**技术选型**：React + TypeScript + Three.js + Zustand（或Redux）

**主要子模块**：

- **UploadModule**：拖拽/点击上传，图片压缩，格式校验
- **TaskProgressModule**：进度条、阶段文案、失败重试
- **ViewerModule**：基于Three.js的GLB加载与渲染，支持轨道控制、环境光、多灯光预设
- **ParamPanelModule**：主石克拉、戒臂宽度/厚度、金属类型、宝石颜色等调节
- **TryOnModule**：调用摄像头 + MediaPipe Hand Landmarker，将戒指实时叠加到手指
- **ExportModule**：下载GLB、生成分享链接、截图保存

**与Mobicat’s Jewels的关系**：  
可直接复用其灯光系统、材质切换、试戴逻辑和交互体验，降低前端开发成本。

### 5.2 任务编排服务（Task Orchestrator）

**职责**：

- 维护任务状态机
- 按顺序调度下游服务
- 处理超时、失败重试、降级

**状态机**：

```text
Pending → Preprocessing → Understanding → Generating → Refining → PostProcessing → Completed
                     ↘ Failed（任意阶段可进入，支持重试）
```

**调度策略**：

- 默认流水线顺序执行
- 支持配置「快速模式」（跳过部分精修）与「高质量模式」
- 失败自动重试最多2次，超过后标记Failed并通知前端

### 5.3 图像处理与理解服务

**处理流程**：

1. 去背景（rembg / SAM2）
2. 图像规范化（尺寸、对比度、中心裁剪）
3. 视觉理解（输出结构化JSON）：
   - setting_type（solitaire / trilogy / halo 等）
   - center_cut（round / emerald / oval / pear...）
   - metal_color（yellow_gold / white_gold / rose_gold / platinum）
   - estimated_carat
   - accent_stones 信息（如有）

**输出示例**：

```json
{
  "setting": "trilogy",
  "center_cut": "emerald",
  "metal": "yellow_gold_18k",
  "estimated_carat": 1.52,
  "confidence": 0.87
}
```

### 5.4 3D生成服务

**策略**：

- **快速通道**：TripoSR（秒级到十几秒）
- **质量通道**：Unique3D 或 Hunyuan3D-2（精度更高，耗时更长）
- 根据用户选择或系统负载自动切换

**输入**：处理后的透明背景图片 + 理解结果（可选作为提示）  
**输出**：原始网格（obj/glb）+ 基础贴图

### 5.5 参数化对齐与精修服务

**核心作用**：解决纯AI重建结构不稳定、爪位不准、比例失调的问题。

**主要步骤**：

1. 根据理解结果匹配最接近的戒托模板
2. 将AI生成的主石与戒臂进行缩放、对齐
3. 用模板约束爪、轨、肩部等关键结构
4. 输出结构更合理的中间模型

### 5.6 后处理服务

- 网格修复（破面、非流形）
- 减面与优化（meshoptimizer）
- UV展开
- 材质分离（金属 vs 宝石）
- 生成PBR材质（baseColor、metallic、roughness、normal）
- 打包标准GLB
- 生成多角度预览图

### 5.7 资产与模板服务

管理：

- 戒托参数化模板库
- 宝石切割库
- 金属PBR材质库
- 预置灯光与环境贴图

---

## 6. 数据设计

### 6.1 核心表结构（PostgreSQL）

**tasks**

```sql
task_id         UUID PRIMARY KEY
user_id         UUID null
status          VARCHAR(32)       -- pending/processing/completed/failed
progress        SMALLINT          -- 0-100
original_image  VARCHAR(512)
result_glb      VARCHAR(512)
preview_image   VARCHAR(512)
understanding   JSONB
error_message   TEXT
created_at      TIMESTAMP
updated_at      TIMESTAMP
finished_at     TIMESTAMP
```

**templates**

```sql
template_id        UUID PRIMARY KEY
name               VARCHAR(128)
setting_type       VARCHAR(64)
supported_cuts     JSONB
parametric_schema  JSONB
base_mesh_url      VARCHAR(512)
thumbnail_url      VARCHAR(512)
status             VARCHAR(16)
```

**materials**

```sql
material_id  UUID PRIMARY KEY
type         VARCHAR(32)       -- metal / gem
name         VARCHAR(64)
pbr_params   JSONB
preview_url  VARCHAR(512)
```

### 6.2 对象存储目录规范

```text
/uploads/{task_id}/original.jpg
/processed/{task_id}/no_bg.png
/generated/{task_id}/raw.glb
/final/{task_id}/model.glb
/final/{task_id}/preview.png
/templates/{template_id}/base.glb
```

---

## 7. 接口设计（核心）

### 7.1 创建任务

```http
POST /api/v1/tasks
Content-Type: multipart/form-data
```

**参数：**

- file: 图片文件
- mode: fast | quality（可选）

**返回：**

```json
{
  "task_id": "xxx",
  "status": "pending"
}
```

### 7.2 查询任务状态

```http
GET /api/v1/tasks/{task_id}
```

**返回：**

```json
{
  "task_id": "xxx",
  "status": "processing",
  "progress": 45,
  "stage": "generating",
  "result": null
}
```

### 7.3 获取最终结果

```http
GET /api/v1/tasks/{task_id}/result
```

**返回：**

```json
{
  "glb_url": "...",
  "preview_url": "...",
  "understanding": { ... },
  "editable_params": { ... }
}
```

### 7.4 参数二次编辑（轻量）

```http
POST /api/v1/tasks/{task_id}/refine
```

```json
{
  "carat": 1.8,
  "metal": "white_gold_18k",
  "band_width": 2.1
}
```

---

## 8. 技术选型汇总

| 类别     | 推荐方案                            | 备注                       |
| -------- | ----------------------------------- | -------------------------- |
| 前端     | React + Three.js + TypeScript       | 可基于Mobicat’s Jewels改造 |
| 后端框架 | FastAPI                             | 与Python AI生态兼容        |
| 任务队列 | ARQ / Celery + Redis                | 轻量优先选ARQ              |
| 数据库   | PostgreSQL + Redis                  |                            |
| 对象存储 | Cloudflare R2 / 阿里云OSS           |                            |
| 去背景   | rembg / SAM2                        |                            |
| 图像理解 | CLIP + 分类模型 / 多模态大模型      |                            |
| 3D生成   | TripoSR + Unique3D / Hunyuan3D      | 双通道                     |
| 网格处理 | trimesh + meshoptimizer + pymeshlab |                            |
| 手部追踪 | MediaPipe Tasks Vision              | 前端本地运行               |
| 部署     | Docker + GPU节点                    | 推理与业务分离             |
| 监控     | Prometheus + Grafana + Sentry       |                            |

---

## 9. 部署架构

### 9.1 逻辑部署

- **业务集群**：CPU容器，水平扩展（Upload、Orchestrator、Query等）
- **AI推理集群**：GPU节点，独立扩缩容，通过内部API调用
- **存储**：对象存储 + CDN
- **缓存与队列**：Redis集群
- **数据库**：PostgreSQL主从

### 9.2 环境建议

| 环境     | 说明                                       |
| -------- | ------------------------------------------ |
| 开发环境 | 单机Docker Compose，可用CPU版模型或远程GPU |
| 测试环境 | 小规格GPU，完整链路                        |
| 生产环境 | 业务与推理分离，GPU按量扩缩容              |

---

## 10. 风险与应对策略

| 风险             | 影响               | 应对措施                                 |
| ---------------- | ------------------ | ---------------------------------------- |
| 单图信息不足     | 结构不准、背面缺失 | 参数化模板兜底 + 用户可调节              |
| 生成耗时过长     | 体验差             | 快速/高质量双通道 + 进度反馈             |
| GPU资源昂贵      | 成本高             | 队列限流、模型量化、闲时缩容             |
| 珠宝工艺细节不足 | 无法直接生产       | 明确交付物为「设计预览级」，后续人工精修 |
| 模型效果波动     | 结果不稳定         | 增加置信度阈值，低置信度走模板优先策略   |

---

## 11. 迭代路线图

### Phase 1：MVP（4–6周）

- 上传 → 去背景 → TripoSR生成 → 基础Three.js查看器
- 任务异步化与进度查询
- 简单结果展示与下载

### Phase 2：可控性增强（4周）

- 款式与材质识别
- 参数化模板对齐
- 金属/宝石切换
- 基础参数调节

### Phase 3：体验完善（4周）

- 虚拟试戴
- 分享链接
- 多灯光与高级材质
- 导出优化

### Phase 4：生产级能力

- 多图/视频输入
- 与工厂打样系统对接
- 价格估算引擎
- 用户体系与历史记录

---

## 12. 附录

### 12.1 参考开源项目

- Mobicat’s Jewels（交互与试戴参考）
- TripoSR
- Unique3D
- Hunyuan3D
- MediaPipe Hand Landmarker
- rembg / SAM

### 12.2 后续可输出文档

- 接口详细文档（OpenAPI）
- 数据库完整DDL
- 前端目录结构与组件设计
- Phase 1 开发任务拆分与排期
- GPU推理服务部署手册
