# ABAP Security Vulnerabilities Reference

> **来源整合自以下权威渠道**：
> - SAP ABAP Keyword Documentation（`help.sap.com`，官方语言帮助）
> - SAP Community Blog: *How to Protect Your ABAP Code Against SQL Injection*
> - SAP Code Vulnerability Analyzer（CVA）检查类别
> - RedRays ABAP Security Scanner 164条规则分类（实际生产系统漏洞统计）
> - CVE-2025-0063、CVE-2025-42957 官方披露
> - DSAG ABAP Development Recommendations
>
> **用途**: 本文件作为 Agent 在 `[SEC]` 和 `[AUTH]` 维度进行代码审查时的权威参考基准

---

## 分类总览

| 类别 | 代码 | 严重程度范围 |
|------|------|------------|
| SQL 注入 | SEC-SQL | CRITICAL |
| 代码注入 | SEC-CODE | CRITICAL |
| OS 命令注入 | SEC-OS | CRITICAL |
| 文件路径攻击 | SEC-FILE | HIGH–CRITICAL |
| RFC/网络安全 | SEC-RFC | HIGH–CRITICAL |
| 授权缺失 | AUTH-MISS | HIGH–CRITICAL |
| 授权绕过 | AUTH-BYP | CRITICAL |
| 硬编码凭证 | SEC-CRED | HIGH |
| 敏感数据泄露 | SEC-DATA | HIGH |
| Web 层漏洞 | SEC-WEB | HIGH |

---

## SEC-SQL：SQL 注入

### 原理

ABAP Open SQL 允许动态构造查询条件。当动态部分包含**来自外部的用户输入**且未经验证时，攻击者可注入恶意 SQL 改变查询逻辑、访问未授权数据，或修改数据库内容。

*SAP ABAP Keyword Documentation 明确指出：*
> "If a dynamic WHERE condition originates in full or in part from outside the program, users could potentially access data for which they usually do not have authorization."

### 漏洞模式

**[SEC-SQL-1] 动态 WHERE 子句拼接用户输入** — 🔴 CRITICAL
```abap
" ❌ 危险代码
PARAMETERS: p_name TYPE string.
DATA(cond) = `NAME = '` && p_name && `'`.
SELECT * FROM scustom WHERE (cond) INTO TABLE @DATA(results).
" 注入示例：p_name = "' OR 1=1 --" → 返回全表数据
```

**[SEC-SQL-2] EXEC SQL（Native SQL）使用外部输入** — 🔴 CRITICAL
```abap
" ❌ 危险代码
EXEC SQL.
  SELECT * FROM zuser WHERE name = :p_name
END-EXEC.
" Native SQL 不受 Open SQL 的参数化保护
```

**[SEC-SQL-3] ADBC（CL_SQL_STATEMENT）动态查询** — 🔴 CRITICAL
```abap
" ❌ 危险代码
DATA(sql) = NEW cl_sql_statement( ).
DATA(query) = `SELECT * FROM zdata WHERE id = '` && p_id && `'`.
sql->execute_query( query ).

" ✅ 正确：使用参数化绑定
sql->set_param( REF #( p_id ) ).
sql->execute_query( `SELECT * FROM zdata WHERE id = ?` ).
```

**[SEC-SQL-4] AMDP/SQLScript 中的 EXEC** — 🔴 CRITICAL
```abap
" ❌ 危险代码（AMDP 方法体中）
EXEC 'UPDATE sflight SET seatsocc = seatsocc + ' || :seats;
" 攻击示例：seats = '2, seatsmax = seatsmax + 999'
```

**[SEC-SQL-5] 动态 FROM 子句（表名来自外部）** — 🔴 CRITICAL
```abap
" ❌ 危险代码 — 表名来自用户输入
SELECT * FROM (p_tabname) INTO TABLE @DATA(results).
" 攻击者可访问任意表，包括 USR02（密码哈希表）
```

### 官方修复方法

使用 `CL_ABAP_DYN_PRG` 类进行输入验证和转义：

```abap
" 方法1：QUOTE() — 为字符串值添加引号并转义单引号
DATA(safe_name) = cl_abap_dyn_prg=>quote( p_name ).
DATA(cond) = `NAME = ` && safe_name.

" 方法2：QUOTE_STR() — 返回带引号的字符串（高版本）
DATA(safe_cond) = `NAME = ` && cl_abap_dyn_prg=>quote_str( p_name ).

" 方法3：CHECK_TABLE_NAME_STR() — 验证表名是否在白名单内
cl_abap_dyn_prg=>check_table_name_str(
  val     = p_tabname
  packages = VALUE #( ( `ZPACKAGE` ) ) ).

" 方法4：CHECK_WHITELIST_STR() — 字段名白名单验证
cl_abap_dyn_prg=>check_whitelist_str(
  val       = p_field
  whitelist = `MATNR,WERKS,BUKRS` ).

" 方法5：使用 @变量绑定代替字面量拼接（推荐优先）
DATA(cond) = `NAME = @p_name`.  " p_name 作为参数，不作为SQL代码
SELECT * FROM scustom WHERE (cond) INTO TABLE @DATA(results).
```

### 检查点速查

在审查代码时，搜索以下关键词并逐一评估：
- `WHERE (` — 括号内是否为变量（动态条件）？
- `EXEC SQL` — 是否有外部输入参与？
- `cl_sql_statement` — 是否使用了 `set_param` 参数化？
- `GENERATE SUBROUTINE POOL` — 来源是否经过验证？
- `INSERT REPORT` — 是否有 S_DEVELOP 权限控制？

---

## SEC-CODE：代码注入

**[SEC-CODE-1] GENERATE SUBROUTINE POOL + 外部输入** — 🔴 CRITICAL
```abap
" ❌ 危险 — 动态生成并执行 ABAP 代码
DATA(code) = p_user_input.  " 来自用户
GENERATE SUBROUTINE POOL code NAME prog.
" CVE-2025-42957：攻击者可执行任意 ABAP，创建用户并赋 SAP_ALL
```

**[SEC-CODE-2] INSERT REPORT 无授权检查** — 🔴 CRITICAL
```abap
" ❌ 危险
INSERT REPORT p_progname FROM lt_source.
" 应确保有 S_DEVELOP OBJTYPE='PROG' ACTVT='01' 的权限检查
```

**[SEC-CODE-3] 动态 CALL FUNCTION/METHOD/PERFORM** — 🟠 HIGH
```abap
" ❌ 危险 — 函数名来自外部
CALL FUNCTION p_funcname EXPORTING ...
CALL METHOD (p_classname)=>(p_methodname).
PERFORM (p_formname) IN PROGRAM (p_progname).
```

---

## SEC-OS：OS 命令注入

**[SEC-OS-1] CALL 'SYSTEM' 含外部输入** — 🔴 CRITICAL
```abap
" ❌ 危险
DATA(cmd) = p_command.  " 用户输入
CALL 'SYSTEM' ID 'COMMAND' FIELD cmd.
```

**[SEC-OS-2] SXPG_COMMAND_EXECUTE 未验证输入** — 🔴 CRITICAL
```abap
" ❌ 危险
CALL FUNCTION 'SXPG_COMMAND_EXECUTE'
  EXPORTING commandname = p_cmd    " 应使用预定义命令，不接受自由输入
            additional_parameters = p_params.
```

**[SEC-OS-3] SXPG_CALL_SYSTEM** — 🔴 CRITICAL

同上，所有 SXPG_* 系列函数调用都应检查输入来源。

---

## SEC-FILE：文件路径攻击

**[SEC-FILE-1] OPEN DATASET 路径含外部输入** — 🟠 HIGH–🔴 CRITICAL
```abap
" ❌ 危险 — 路径遍历风险
DATA(path) = '/data/' && p_filename.
OPEN DATASET path FOR INPUT IN TEXT MODE.
" 攻击示例：p_filename = '../../etc/passwd'
```

**[SEC-FILE-2] 未检查 SY-SUBRC** — 🟠 HIGH
```abap
" ❌ 危险 — 文件操作后不检查结果
OPEN DATASET lv_path FOR INPUT IN TEXT MODE ENCODING DEFAULT.
READ DATASET lv_path INTO lv_line.  " 文件可能未成功打开

" ✅ 正确
OPEN DATASET lv_path FOR INPUT IN TEXT MODE ENCODING DEFAULT.
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE cx_file_not_found.
ENDIF.
```

**[SEC-FILE-3] 推荐使用逻辑文件名** — 🟢 LOW
```abap
" ✅ 使用事务 FILE 定义逻辑文件名，通过 FILE_GET_NAME 转换
CALL FUNCTION 'FILE_GET_NAME'
  EXPORTING logical_filename = 'Z_REPORT_OUTPUT'
  IMPORTING file_name        = lv_physical_path.
```

---

## SEC-RFC：RFC 与网络安全

**[SEC-RFC-1] RFC 目标来自外部输入** — 🔴 CRITICAL
```abap
" ❌ 危险 — RFC 目标可被操控
CALL FUNCTION 'Z_SENSITIVE_FM' DESTINATION p_dest.
" 攻击者可将 p_dest 指向恶意系统
```

**[SEC-RFC-2] RFC-enabled FM 缺少授权检查** — 🔴 CRITICAL
```abap
" ❌ 危险 — 无授权检查的 RFC FM
FUNCTION z_delete_records.
  " *"  REMOTE-ENABLED MODULE
  DELETE FROM zdata WHERE id = iv_id.  " 任何 RFC 调用者均可执行
ENDFUNCTION.
```

**[SEC-RFC-3] RFC FM 中使用 MESSAGE TYPE 'A'/'I'/'W'** — 🟠 HIGH
```abap
" ❌ 危险 — 在 RFC FM 中使用对话消息会导致 RFC 调用崩溃
FUNCTION z_rfc_fm.
  " *"  REMOTE-ENABLED MODULE
  IF error.
    MESSAGE 'Error occurred' TYPE 'E'.  " 会抛出运行时异常且无法被 RFC 客户端捕获
  ENDIF.
ENDFUNCTION.
" ✅ 应改用 EXCEPTIONS 参数传递错误
```

**[SEC-RFC-4] DESTINATION 'NONE' 授权绕过** — 🔴 CRITICAL
```abap
" ❌ 危险 — DESTINATION 'NONE' 在某些配置中绕过授权传播
CALL FUNCTION 'SENSITIVE_BAPI' DESTINATION 'NONE'.
```

---

## AUTH-MISS：授权缺失

### 原则

*SAP 明确规定：SAP 数据库层对自定义代码不执行授权检查。*
> "SAP doesn't enforce authorization checks at the database level for custom code."

因此，**每个读取或修改敏感数据的操作**都需要显式的 AUTHORITY-CHECK。

**[AUTH-MISS-1] 读取业务敏感表前无权限检查** — 🔴 CRITICAL

高风险表（必须检查）：

| 表 | 业务含义 | 建议权限对象 |
|----|---------|------------|
| BKPF / BSEG | FI 凭证 | F_BKPF_BUK, F_BKPF_KOA |
| MKPF / MSEG | MM 物料移动 | M_MSEG_BWA, M_MSEG_WMB |
| VBAK / VBAP | SD 销售订单 | V_VBAK_AAT, V_VBAK_VKO |
| EKKO / EKPO | MM 采购订单 | M_BEST_BSA, M_BEST_EKG |
| PA* / HRP* | HR 人事数据 | P_ORGIN, P_PERNR |
| USR* | 用户主数据 | S_USER_GRP |
| T001 / T001W | 公司代码/工厂配置 | — |

```abap
" ❌ 危险 — 直接读取 FI 凭证无授权检查
SELECT * FROM bkpf INTO TABLE @DATA(docs) WHERE bukrs = @p_bukrs.

" ✅ 正确
AUTHORITY-CHECK OBJECT 'F_BKPF_BUK'
  ID 'BUKRS' FIELD p_bukrs
  ID 'ACTVT' FIELD '03'.  " 03 = Display
IF sy-subrc <> 0.
  MESSAGE 'No authorization' TYPE 'E'.
ENDIF.
SELECT * FROM bkpf INTO TABLE @DATA(docs) WHERE bukrs = @p_bukrs.
```

**[AUTH-MISS-2] AUTHORITY-CHECK 后未检查 SY-SUBRC** — 🔴 CRITICAL
```abap
" ❌ 危险 — 权限检查形同虚设
AUTHORITY-CHECK OBJECT 'F_BKPF_BUK'
  ID 'BUKRS' FIELD bukrs
  ID 'ACTVT' FIELD '02'.
" 忘记检查 SY-SUBRC！任何用户都能继续执行
UPDATE bkpf SET ...
```

**[AUTH-MISS-3] CALL TRANSACTION 前无权限检查** — 🟠 HIGH
```abap
" ❌ 危险
CALL TRANSACTION 'FB01' USING bdcdata.

" ✅ 正确
AUTHORITY-CHECK OBJECT 'S_TCODE' ID 'TCD' FIELD 'FB01'.
IF sy-subrc <> 0. RETURN. ENDIF.
CALL TRANSACTION 'FB01' USING bdcdata.
```

**[AUTH-MISS-4] SUBMIT 前无权限检查** — 🟠 HIGH
```abap
" ❌ 危险 — 直接提交报表执行
SUBMIT zmmr0002 VIA JOB 'BATCH' NUMBER jobno AND RETURN.

" ✅ 应检查 S_PROGRAM 或相关授权对象
```

---

## AUTH-BYP：授权绕过

**[AUTH-BYP-1] 用 SY-UNAME 直接控制访问** — 🔴 CRITICAL
```abap
" ❌ 危险 — 硬编码用户名作为"超级用户"
IF sy-uname = 'ADMIN' OR sy-uname = 'BASIS'.
  " 跳过所有检查
  SKIP_AUTHORIZATION = abap_true.
ENDIF.
" SY-UNAME 可被伪造；应使用 AUTHORITY-CHECK
```

**[AUTH-BYP-2] 基于系统ID或客户端的条件分支** — 🔴 CRITICAL
```abap
" ❌ 危险 — 生产系统中可能被激活的后门
IF sy-sysid = 'PRD' AND sy-mandt = '100'.
  AUTHORITY-CHECK OBJECT '...'
ELSE.
  " 开发环境跳过所有权限检查 — 但如果代码到达生产环境？
ENDIF.
```

**[AUTH-BYP-3] 权限对象使用通配符 '*'** — 🟠 HIGH
```abap
" ❌ 危险 — 使用通配符规避精确检查
AUTHORITY-CHECK OBJECT 'M_MSEG_BWA'
  ID 'BWART' FIELD '*'    " * 表示所有移动类型，过于宽泛
  ID 'ACTVT' FIELD '01'.
```

**[AUTH-BYP-4] 动态权限对象名** — 🟠 HIGH
```abap
" ❌ 危险 — 权限对象名来自变量
DATA(auth_obj) = p_object.
AUTHORITY-CHECK OBJECT (auth_obj) ...
```

---

## SEC-CRED：硬编码凭证

**[SEC-CRED-1] 源码中的密码/密钥** — 🟠 HIGH
```abap
" ❌ 危险
DATA(password) = 'P@ssw0rd123'.
DATA(api_key)  = 'sk-abcdef1234567890'.
DATA(conn_str) = 'user=admin;pwd=secret'.

" ✅ 使用 SAP Secure Storage
CALL FUNCTION 'SUSR_USER_AUTH_FOR_RFC_GET'
  " 或使用 SM59 RFC 目标中的安全凭证存储
```

**[SEC-CRED-2] 特定用户名硬编码** — 🟠 HIGH
```abap
" ❌ 危险
CALL FUNCTION 'Z_RFC_FM'
  EXPORTING username = 'BATCHUSER01'
             password = 'Initial1'.
```

---

## SEC-DATA：敏感数据泄露

**[SEC-DATA-1] 个人数据写入 Spool 或文件未脱敏** — 🟠 HIGH
```abap
" ❌ 危险 — 身份证号、银行账号直接输出
WRITE: employee-id_number.
WRITE: vendor-bank_account.

" ✅ 应在输出时脱敏
DATA(masked) = '****' && employee-id_number+12(4).
```

**[SEC-DATA-2] 调试信息包含敏感数据** — 🟢 LOW
```abap
" ❌ 避免
MESSAGE |Debug: user={ sy-uname } pwd={ lv_password }| TYPE 'I'.
```

---

## SEC-WEB：Web 层漏洞

> 适用于 BSP、Web Dynpro、ICF Handler、OData DPC Extension 等场景

**[SEC-WEB-1] 输出未转义（XSS）** — 🟠 HIGH
```abap
" ❌ 危险 — BSP 中直接输出用户输入
response->append_cdata( p_input ).

" ✅ 应进行 HTML 转义
CALL FUNCTION 'ESCAPE_HTML_CHARS'
  EXPORTING unescaped = p_input
  IMPORTING escaped   = lv_safe_output.
```

**[SEC-WEB-2] OData DPC Extension 无后端权限检查** — 🟠 HIGH
```abap
" ❌ 危险 — 仅依赖 Fiori 前端隐藏不代表有安全保护
METHOD zorder_dpc_ext=>orderitem_get_entityset.
  " 无 AUTHORITY-CHECK，任何能访问 OData 服务的用户均可读取
  SELECT * FROM vbap INTO TABLE et_entityset.
ENDMETHOD.
```

---

## 检查优先级速查矩阵

当审查代码时，按以下顺序扫描关键词：

```
优先级 1 — 立即检查（CRITICAL 风险）：
  EXEC SQL          → SEC-SQL-2
  GENERATE SUBROUTINE POOL → SEC-CODE-1
  INSERT REPORT     → SEC-CODE-2
  CALL 'SYSTEM'     → SEC-OS-1
  SXPG_*            → SEC-OS-2/3
  WHERE (           → SEC-SQL-1（动态条件）
  DESTINATION (     → SEC-RFC-1（动态目标）

优先级 2 — 重要检查（HIGH 风险）：
  AUTHORITY-CHECK   → 检查后是否有 SY-SUBRC 校验
  SELECT ... WHERE ( → 动态 WHERE 是否安全
  OPEN DATASET      → 路径来源，SY-SUBRC 检查
  cl_sql_statement  → 是否用了 set_param
  CALL FUNCTION '...' DESTINATION → RFC 目标来源
  SY-UNAME =        → AUTH-BYP-1

优先级 3 — 常规检查（MEDIUM/LOW）：
  CALL FUNCTION ... EXCEPTIONS → 是否有穷举 + SY-SUBRC
  MESSAGE TYPE      → 在 RFC FM 中禁止对话消息
  '密码'/'password'/'key'/'token' 字面量 → SEC-CRED
```

---

## 相关 CVE 参考

| CVE | 影响 | 模式 | 对应规则 |
|-----|------|------|---------|
| CVE-2025-0063 | SQL 注入（CVSS 9.9） | RFC FM + Informix EXEC SQL + 无输入验证 | SEC-SQL-2 |
| CVE-2025-42957 | 代码注入（/SLOAE/DEPLOY） | RFC FM 参数验证不足 → GENERATE SUBROUTINE POOL | SEC-CODE-1 |
| CVE-2025-30012 | 反序列化 RCE | CL_ABAP_SERIALIZER 信任不可信字节流 | SEC-CODE |

---

## 工具参考

| 工具 | 覆盖范围 | 可用性 |
|------|---------|-------|
| SAP Code Vulnerability Analyzer (CVA) | SQL注入、代码注入等 | 需额外许可证 |
| ABAP Test Cockpit (ATC) | 基础代码质量 | 标准包含（不含安全检查） |
| code-pal-for-ABAP | Clean ABAP 规则 | 开源免费 |
| RedRays ABAP Scanner | 164条安全规则 | 商业工具 |
| abaplint | 语法和最佳实践 | 开源免费（CI/CD集成） |

---
*本文件整合自 SAP 官方文档、安全社区最佳实践及 CVE 披露，供 AI Agent 静态代码审查使用。*
*动态运行时行为、配置依赖的授权及传输环境差异超出本文件覆盖范围。*
