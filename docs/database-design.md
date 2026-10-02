# 第一版 SQLite 数据库设计

## 1. 目标与范围

承接 architecture.md 的单用户、本地运行方案，支持个人资料、职位整理、匹配分析、申请材料和投递跟踪。本文件只定义数据结构和规则，不包含 SQL、程序代码或迁移脚本，也不创建数据库文件。

第一版使用一个数据库文件：data/ai-job-agent.db。设计六张业务表，不引入账号、权限、公司主数据、任务队列或通用版本管理系统。

## 2. 目录与存储职责

| 目录 | 用途 |
| --- | --- |
| profile/ | 用户导入的个人资料附件；可检索、可编辑的正文存入数据库 |
| jobs/ | 可选的职位原始附件；职位正文及字段存入数据库 |
| resume/ | 简历和求职信导出文件；草稿正文与版本存入数据库 |
| data/ | SQLite 数据库及本地备份 |

数据库是业务记录的唯一事实来源，不在目录中维护第二套职位或申请状态。第一版不依赖附件才能完成流程；附件只用于保留原件或导出材料。

这些目录仅提交 .gitkeep 占位文件。真实资料、导出文件、数据库和备份不进入 Git；AI 密钥不存入业务表。

## 3. 通用约定

- 主键使用 INTEGER；外键使用相同类型，不需要分布式 ID。
- 正文、枚举和 JSON 使用 TEXT。必填文本应拒绝空白内容；未知字段使用 NULL，不用虚构值补齐。
- created_at、updated_at 等时间使用统一的 UTC ISO 8601 文本格式，精确到毫秒；界面转换为用户当地时间。follow_up_date 使用当地日历日期 YYYY-MM-DD。
- 表格标注“必填”的字段落实为非空约束；有限取值和正整数范围落实为检查约束。格式、文本非空白与业务规则同时在应用层校验。
- JSON 字段存合法 JSON，由应用校验结构。第一版不依赖 JSON 查询或索引；需要筛选的字段单独设列。
- 每个数据库连接都显式启用并检查外键约束，不能只声明关系。[SQLite 外键说明](https://www.sqlite.org/foreignkeys.html)
- SQLite 的日期时间可用 TEXT 保存；本文采用一种固定格式以便排序和比较。[SQLite 类型说明](https://www.sqlite.org/datatype3.html)

## 4. 表结构

### 4.1 profiles：当前个人资料

单用户只保存一条当前资料，id 固定为 1，并通过主键及取值约束保证最多一条。历史输入保存在分析与草稿的快照中。

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| id | INTEGER | 是 | 主键，固定为 1 |
| resume_text | TEXT | 是 | 用户确认的基础简历正文 |
| skills_text | TEXT | 否 | 技能补充 |
| experience_text | TEXT | 否 | 工作、教育及项目经历补充 |
| preferences_text | TEXT | 否 | 目标职位、城市、薪资等偏好，第一版允许自由文本 |
| revision | INTEGER | 是 | 初始为 1；内容或偏好修改时递增 |
| created_at | TEXT | 是 | 创建时间 |
| updated_at | TEXT | 是 | 最近修改时间 |

AI 不直接修改此表；用户确认后才能更新事实信息。未录入资料时仍可导入职位和维护申请记录。

### 4.2 jobs：职位

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| id | INTEGER | 是 | 主键 |
| title | TEXT | 否 | 职位名称，提取失败时可暂缺 |
| company | TEXT | 否 | 公司名称 |
| location | TEXT | 否 | 工作地点 |
| salary_text | TEXT | 否 | 原始薪资描述，不强行换算成年薪 |
| source_url | TEXT | 否 | 用户提供的来源链接 |
| raw_text | TEXT | 是 | 导入的职位原文，不被 AI 提取结果覆盖 |
| requirements_text | TEXT | 否 | 提取并允许用户修正的职位要求 |
| revision | INTEGER | 是 | 初始为 1；原文或职位字段修改时递增 |
| archived_at | TEXT | 否 | 归档时间；NULL 表示未归档 |
| created_at | TEXT | 是 | 导入时间 |
| updated_at | TEXT | 是 | 最近修改时间 |

重复检测先比较来源链接，再提示公司和职位名称相同的记录。两者都不设唯一约束，因为相同链接可能复用，同名职位也可能不同；由用户决定保留或更新。修改原文须由用户明确执行。

### 4.3 match_analyses：匹配分析

每次成功分析新增一条记录，历史结果不覆盖。只保存完成校验的结果，失败请求不产生半成品业务记录。

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| id | INTEGER | 是 | 主键 |
| job_id | INTEGER | 是 | 外键，关联 jobs.id |
| profile_id | INTEGER | 是 | 外键，关联 profiles.id |
| profile_revision | INTEGER | 是 | 分析时的资料版本 |
| job_revision | INTEGER | 是 | 分析时的职位版本 |
| input_snapshot_json | TEXT | 是 | 当次实际输入：资料、偏好、职位内容及对应版本 |
| matches_json | TEXT | 是 | 匹配点列表；每项包含职位要求与个人经历依据 |
| gaps_json | TEXT | 是 | 差距列表及依据 |
| questions_json | TEXT | 是 | 待确认信息列表；无内容时为空数组 |
| summary_text | TEXT | 是 | 总体建议，不输出缺乏依据的精确分数 |
| model_name | TEXT | 是 | 服务及模型标识 |
| prompt_version | TEXT | 是 | 提示词版本 |
| created_at | TEXT | 是 | 分析完成时间 |

快照保留生成所需且经过最小化处理的输入，不额外复制无关联系方式。当前 revision 与记录中的版本不同，就在界面提示分析已过期；无需保存容易失同步的 is_stale 字段。

### 4.4 material_drafts：申请材料版本

简历调整稿和求职信共用一张表。每次重新生成或用户保存修改，都新增一个版本；确认状态可更新，但正文不能原地改写。

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| id | INTEGER | 是 | 主键 |
| job_id | INTEGER | 是 | 外键，关联 jobs.id |
| profile_id | INTEGER | 是 | 外键，关联 profiles.id |
| material_type | TEXT | 是 | resume 或 cover_letter |
| version | INTEGER | 是 | 同一职位、同一材料类型内从 1 递增 |
| content_text | TEXT | 是 | 该版本完整正文 |
| origin | TEXT | 是 | ai 或 manual，表示当前版本由谁生成或修改 |
| profile_revision | INTEGER | 是 | 本版本使用的资料版本 |
| job_revision | INTEGER | 是 | 本版本使用的职位版本 |
| input_snapshot_json | TEXT | 是 | 当次依据；编辑旧稿时保留原稿的输入快照和版本，不冒充最新资料 |
| model_name | TEXT | 否 | AI 生成时必填；人工版本允许为空 |
| prompt_version | TEXT | 否 | AI 生成时必填；人工版本允许为空 |
| confirmed_at | TEXT | 否 | 用户核实该版本的时间；NULL 表示待确认 |
| created_at | TEXT | 是 | 本版本保存时间 |

组合 (job_id, material_type, version) 唯一，version 必须为正整数。修改已确认稿会产生一个未确认的新版本。使用旧分析起草时，保存其中的依据并校验版本；资料已变化时先提示用户重新检查。

导出文件是草稿的派生副本，第一版不设置文件表或导出路径字段。实际投递版本由申请记录关联，避免“最新草稿”与“已投递材料”混淆。

### 4.5 applications：申请记录

第一版每个职位最多一条申请记录；一条记录贯穿准备、投递和后续面试。重复申请同一职位的独立轮次暂不支持。

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| id | INTEGER | 是 | 主键 |
| job_id | INTEGER | 是 | 外键，关联 jobs.id；唯一 |
| status | TEXT | 是 | 当前状态，默认 preparing |
| submitted_resume_id | INTEGER | 否 | 外键，关联实际投递的 material_drafts.id |
| submitted_cover_letter_id | INTEGER | 否 | 外键，关联实际投递的 material_drafts.id |
| submitted_at | TEXT | 否 | 用户记录的实际投递时间；确认已投递时必填 |
| notes_text | TEXT | 否 | 申请备注 |
| follow_up_date | TEXT | 否 | 下一次跟进日期 |
| created_at | TEXT | 是 | 记录创建时间 |
| updated_at | TEXT | 是 | 最近修改时间 |

状态仅允许 preparing（准备中）、submitted（已投递）、interviewing（面试中）、offered（录用）、rejected（拒绝）、closed（已关闭）。拒绝和关闭可能发生在投递前，因此不强制这两种状态有 submitted_at。

提交申请时，关联材料须属于同一职位、具有正确 material_type 且已由用户确认；由应用在事务中校验。允许用户在外部编辑和投递材料，此时两个关联字段可为空，备注说明即可。生成或导出材料不自动更新申请状态。

### 4.6 application_status_history：状态历史

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| id | INTEGER | 是 | 主键 |
| application_id | INTEGER | 是 | 外键，关联 applications.id |
| from_status | TEXT | 否 | 上一状态；创建申请的首条记录为空 |
| to_status | TEXT | 是 | 新状态；取值范围与 applications.status 一致 |
| note_text | TEXT | 否 | 本次变化原因或更正说明 |
| created_at | TEXT | 是 | 变化记录时间 |

创建申请时写入首条历史。此后每次状态变化追加记录，不覆盖或删除旧历史；更正状态也追加记录并写明原因。同一时间的记录按 id 排序。

## 5. 关系、约束与索引

### 关系

- 一条个人资料对应多条匹配分析和材料版本。
- 一个职位对应多条匹配分析、多个材料版本和至多一条申请。
- 一条申请对应多条状态历史，并可引用已投递的两个材料版本。

所有外键采用限制删除，不自动级联删除求职记录。日常移除职位采用归档，终止申请采用 closed。用户主动清除数据时，在事务中按依赖顺序删除，材料附件另行处理；归档不等于删除个人信息。

### 首版索引

| 表 | 列 | 类型与用途 |
| --- | --- | --- |
| jobs | source_url | 普通索引，辅助重复提示 |
| applications | job_id | 唯一约束，防止重复申请记录 |
| applications | status, follow_up_date | 组合索引，筛选状态及到期待办 |
| match_analyses | job_id, created_at | 组合索引，查询职位的分析历史 |
| material_drafts | job_id, material_type, version | 唯一约束，同时支持按职位和类型取版本 |
| application_status_history | application_id, id | 组合索引，顺序查看状态历史 |

主键自带的索引不重复创建。第一版不对简历全文、JSON、每个外键或每个时间列单独建索引；出现实际查询瓶颈后再调整。

## 6. 写入与一致性

- 更新资料或职位内容时，将正文修改、revision 递增和 updated_at 更新放在同一事务。
- AI 请求在数据库事务外执行。开始请求前读取一致的输入快照；返回后校验结果，用短事务保存。期间资料变化不会改变快照，结果显示为旧版本。
- 申请创建与首条历史同时提交；申请状态更新与历史追加也同时提交，失败时一起回滚。历史中的 from_status 必须等于更新前的状态。
- 保存材料新版本时，在同一写事务内读取下一版本号并插入，结合唯一约束防止重复版本。
- 修改备注或跟进日期不产生状态历史，只更新申请记录的 updated_at。
- 不在数据库写事务内等待网络、AI 或用户确认。失败显示给用户，保留已有数据，允许人工重试。

## 7. 备份与后续实现

数据库结构版本采用 SQLite 自带的 user_version 元数据，第一版为 1，不另建迁移管理表。后续结构调整前先备份，再按顺序升级；迁移脚本留到实现阶段。

备份应包含数据库和四个目录中的实际资料。运行中备份使用 SQLite 提供的一致性备份机制；不能直接复制正在写入的数据库文件。文件附件备份期间暂停相关写入，恢复后检查记录与文件一致性。[SQLite 备份说明](https://www.sqlite.org/backup.html)

后续实现验收应覆盖：重启后数据完整、外键拒绝孤立记录、重复申请被拒绝、状态与历史一起回滚、修改资料后旧结果提示过期、已投递材料版本可追溯，以及从备份实际恢复成功。

暂不增加独立公司表、技能标签表、提醒任务表或失败请求表。需要结构化薪资筛选、多轮申请或附件管理时，再针对明确需求扩展。
