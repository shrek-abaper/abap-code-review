---
name: abap-security-review
description: "对 SAP ABAP 程序进行发布前安全与风险评估，输出正式评估报告。Use this skill whenever the user asks to review, audit, assess, or check an ABAP program before release — including phrases like '安全评估'、'风险检查'、'发布审查'、'代码审查'、'review ABAP'、'check program before transport'、'evaluate ABAP for release'. Also trigger when user mentions a program name (e.g. ZMMR0002) alongside words like 'review', 'check', 'safe to release', or '能不能上线'. Always use this skill — do not attempt ABAP code review without it."
---

# ABAP Security Review Skill

对 SAP ABAP 程序进行发布前安全与风险评估，输出可流转的正式 Markdown 报告。

## Trigger Phrase

```
参考 abap-security-review SKILL，对程序 [PROGRAM_NAME] 进行安全与风险评估，
Transport [DEVKXXXXXX]，变更说明：[一句话业务目的]。
报告保存至 reports/ 目录。
```

如需功能完整性检查（[FUNC] 维度），附上需求文档路径或在对话中描述需求。

---

## Step 0 — Load References First

在读取任何 ABAP 代码之前，必须先按顺序加载以下两个文件，不得加载 references/ 目录下的其他文件：

1. `references/REF_ABAP_SECURITY.md` — [SEC] [AUTH] [INTERFACE] 维度判断依据
2. `references/REF_CLEAN_ABAP.md` — [STD] 维度判断依据

> 每个 [SEC] / [AUTH] / [STD] 维度的 finding 必须引用对应规则编号（如 `SEC-SQL-1`、`[E-2]`）。
> 无规则编号支撑的 finding 视为无效，必须删除。

---

## Step 1 — Read Source Code

使用 ABAP CLI SKILL 读取目标程序的完整源码。

| Object Type | 读取范围 |
|-------------|---------|
| REPORT | 主程序 + 全部 INCLUDE |
| Global Class | 类定义 + 全部 METHOD 实现 |
| Function Module | 目标 FM + 同函数组其他 FM |
| Enhancement / BAdI | Enhancement Spot 定义 + 全部活跃实现 |

读取顺序：主程序/类定义 → INCLUDE（按出现顺序）→ METHOD（逐个）。

无法读取的对象：记入报告 Scope Limitations，标记为"unreviewed"，不得假设其安全。

---

## Step 2 — Analyze (9 Dimensions)

按以下顺序完成全部 9 个维度，不得跳过（无发现则注明"No issues found"）。

### [SEC] Security Vulnerabilities
参考 `REF_ABAP_SECURITY.md`。按优先级扫描：

```
Priority 1（立即扫描）:
  EXEC SQL                    → SEC-SQL-2
  GENERATE SUBROUTINE POOL    → SEC-CODE-1
  INSERT REPORT               → SEC-CODE-2
  CALL 'SYSTEM' / SXPG_*     → SEC-OS-1/2/3
  WHERE ( <变量> )             → SEC-SQL-1
  DESTINATION ( <变量> )       → SEC-RFC-1

Priority 2（重要检查）:
  cl_sql_statement            → 是否用 set_param 参数化
  OPEN DATASET                → 路径来源 + SY-SUBRC
  CALL FUNCTION DESTINATION   → RFC 目标来源
  字面量含 password/key/token  → SEC-CRED-1
```

### [AUTH] Authorization & Access Control
参考 `REF_ABAP_SECURITY.md` → AUTH-MISS-*, AUTH-BYP-*

高风险表（读写前必须有 AUTHORITY-CHECK）：BKPF/BSEG、MKPF/MSEG、VBAK/VBAP、EKKO/EKPO、PA*/HRP*

检查：① 敏感表操作前有 AUTHORITY-CHECK → ② CHECK 后立即检查 SY-SUBRC → ③ 无 SY-UNAME 硬编码绕过 → ④ RFC FM 有权限检查

> **Special rule**: [SEC] 或 [AUTH] 中任何 HIGH 发现，发布决策等同 CRITICAL（NO-GO）。

### [DATA] Data Integrity & Exception Handling
参考 `REF_CLEAN_ABAP.md` [E-2] [E-3]

检查：CALL FUNCTION 后 SY-SUBRC → READ TABLE / SELECT SINGLE 后 SY-SUBRC → OPEN DATASET 后 SY-SUBRC → 写操作前 ENQUEUE 锁 → COMMIT WORK 不在循环内 → 异常路径有 ROLLBACK WORK → 无空 CATCH 吞掉异常

### [PERF] Performance Risks
参考 `REF_CLEAN_ABAP.md` [T-1] [T-2] [T-3]

检查：SELECT 在 LOOP 内（→ [T-2]，大表时为 CRITICAL）→ SELECT * 无投影（→ [T-3]）→ 全表扫描（无 WHERE）→ LOOP 内 READ TABLE WITH KEY（→ [T-1]，改 SORTED/HASHED）→ 高量表（BKPF/BSEG/MKPF/MSEG）无行数保护

### [STD] ABAP Code Standards
参考 `REF_CLEAN_ABAP.md` [L-*] [C-*] [N-*] [M-*]

检查废弃语句（`[L-3]`）、硬编码业务值 bukrs/werks/mandt（`[C-3]`，企业环境通常 HIGH）、过长方法（`[M-2]`，>20 行标记）、注释掉的代码块（`[CM-3]`）、Unicode 兼容性

### [INTERFACE] Interface & Integration Risks
参考 `REF_ABAP_SECURITY.md` SEC-RFC-*、SEC-WEB-*

检查：RFC FM 内有对话消息 MESSAGE TYPE A/I/W（→ SEC-RFC-3）→ RFC FM 参数有类型定义 → EXCEPTIONS 完整声明 → CALL FUNCTION DESTINATION 超时配置 → OData DPC Extension 有后端权限检查（→ SEC-WEB-2）

### [CHANGE] Change Impact Assessment
评估：受影响数据库表（读/写/删全列）→ 是否修改 SAP 标准对象 → 是否共享其他程序的 Include/FM → Transport 先决条件和跨系统依赖

### [COMP] Compliance & Audit Trail
检查：PII 数据读取/导出有合规依据 → FI/CO 过账有双人控制 → 主数据变更记录至 CDHDR/CDPOS → 程序记录运行者/时间/参数 → 无单人同时发起和审批的 SoD 路径

### [FUNC] Functional Completeness *(可选)*
**仅在用户提供需求规格时执行**，否则在报告中注明跳过原因。

若执行：检查程序入口覆盖业务场景 → 边界条件处理（空值/零值/超大数据集）→ 核心业务逻辑与需求一致 → 计算逻辑精确 → 输出字段完整 → 集成接口数据完整传递

---

## Step 3 — Severity Classification

| Level | Label | Release Impact |
|-------|-------|---------------|
| 🔴 | CRITICAL | Blocks release |
| 🟠 | HIGH | Should block release |
| 🟡 | MEDIUM | Fix in next sprint |
| 🟢 | LOW | Advisory |
| ℹ️ | INFO | No action required |

当不确定严重级别时，选择**更高**的那个。

---

## Step 4 — Release Decision

```
CRITICAL finding 存在             → NO-GO
[SEC]/[AUTH] 中存在 HIGH finding  → NO-GO（special rule）
其他维度存在 HIGH finding          → CONDITIONAL GO（需技术负责人签字接受）
仅 MEDIUM / LOW / INFO            → GO
```

---

## Step 5 — Generate Report

加载 `references/REPORT_TEMPLATE.md`，严格按模板生成报告。

报告语言：标题和规则编号用**英文**，风险说明和修复建议用**中文**。

文件命名：`ABAP_REVIEW_[PROGRAM_NAME]_[YYYYMMDD].md`
有 CRITICAL finding 时前缀：`CRITICAL_ABAP_REVIEW_[PROGRAM_NAME]_[YYYYMMDD].md`

保存至用户指定的 `reports/` 目录，完成后确认文件路径。

---

## Behavior Rules

| Rule | Requirement |
|------|------------|
| References first | Step 0 完成后才能开始 Step 1 |
| Evidence-first | 每个 finding 必须含真实代码片段（≤15 行），无代码证据的 finding 无效 |
| Rule citation | [SEC]/[AUTH]/[STD] finding 必须引用规则编号 |
| No false negatives | 未读取的对象 → "partially reviewed"，不得写"no issues found" |
| No duplication | 同一模式多处出现 → 一个 finding 列全部位置 |
| FUNC gate | 无需求文档 → 跳过并注明，不得自行推断需求 |
