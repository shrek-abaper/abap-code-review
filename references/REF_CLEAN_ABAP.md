# Clean ABAP Rules Reference

> **来源**: SAP 官方开源规范 `github.com/SAP/styleguides/clean-abap/CleanABAP.md`
> **许可**: CC BY 4.0（开放使用，需署名）
> **用途**: 本文件作为 Agent 在 `[STD]` 维度进行代码审查时的官方判断依据
> **版本摘录**: 聚焦对发布评估最有价值的规则，完整规范请参考原始仓库

---

## 1. Names（命名规范）

### 规则优先级说明
Clean ABAP 将命名分为三类严重程度：
- **Critical**：直接影响可读性和维护性，违反应标记为 🟠 HIGH
- **Major**：降低代码可理解性，违反应标记为 🟡 MEDIUM
- **Minor**：细节改进，违反应标记为 🟢 LOW

### 核心规则

**[N-1] 使用描述性名称，而不是缩写** *(Major)*
```abap
" ❌ 反例
DATA: flnm TYPE string.
DATA: cnt TYPE i.

" ✅ 正例
DATA: file_name TYPE string.
DATA: entry_count TYPE i.
```
*官方依据: "Use intention-revealing names. Reveal the purpose."*

**[N-2] 每个概念使用同一个词** *(Major)*
```abap
" ❌ 反例 — 同一含义用了三种写法
" get_data(), fetch_info(), retrieve_records()

" ✅ 正例 — 统一用法
" get_order(), get_delivery(), get_invoice()
```

**[N-3] 类用名词，方法用动词** *(Major)*
```abap
" ❌ 反例
CLASS check IMPLEMENTATION.   " 不明确是什么
METHOD material TYPE string.  " 应该是动词

" ✅ 正例
CLASS material_validator IMPLEMENTATION.
METHOD validate_material_number.
```

**[N-4] 避免编码前缀（匈牙利命名法）** *(Minor)*
```abap
" ❌ 反例 — 旧式匈牙利前缀（仍广泛存在于遗留代码）
DATA: lv_name TYPE string.
DATA: lt_orders TYPE orders_table.
DATA: lo_handler TYPE REF TO cl_handler.

" ✅ SAP Clean ABAP 推荐方向（现代代码）
DATA: name TYPE string.
DATA: orders TYPE orders_table.
DATA: handler TYPE REF TO cl_handler.
```
> ⚠️ **评审注意**：前缀在遗留系统中极为普遍。本条规则**不强制**要求改造，
> 仅在新增代码中混用新旧风格时标记为 LOW。

**[N-5] 避免无意义词汇** *(Major)*
```abap
" ❌ 反例
DATA: data TYPE string.
DATA: info TYPE string.
DATA: flag TYPE abap_bool.  " "flag" 不说明是什么标志

" ✅ 正例
DATA: customer_name TYPE string.
DATA: payment_overdue TYPE abap_bool.
```

---

## 2. Language（语言使用）

**[L-1] 优先使用面向对象而非过程式** *(Major)*
```abap
" ❌ 反例 — 应避免在新代码中使用 FORM
FORM calculate_total USING iv_amount TYPE dmbtr.

" ✅ 正例
CLASS order_calculator DEFINITION.
  METHODS calculate_total IMPORTING amount TYPE dmbtr.
ENDCLASS.
```
*官方依据: "SAP postulates non-OO code modules as obsolete. Use FMs only where classes cannot be used (RFC, update modules)."*

**[L-2] 优先使用函数式语言结构** *(Minor)*
```abap
" ❌ 反例
DATA result TYPE i.
result = 1 + 2.

" ✅ 正例
DATA(result) = 1 + 2.
```

**[L-3] 避免废弃语句** *(Major)*

| 废弃写法 | 现代替代 |
|---------|---------|
| `MOVE a TO b` | `b = a` |
| `COMPUTE x = y + z` | `x = y + z` |
| `WRITE val TO str` | `str = \|{ val }\|` 或类型转换 |
| `REFRESH itab` | `CLEAR itab` |
| `FREE itab` | `CLEAR itab` 或 `FREE itab`（仅需释放内存时）|
| `MULTIPLY x BY y` | `x = x * y` |
| `DIVIDE x BY y` | `x = x / y` |
| `ADD x TO y` | `y = y + x` |

**[L-4] 使用内联声明** *(Minor)*
```abap
" ❌ 反例
DATA result TYPE dmbtr.
SELECT SINGLE amount INTO result FROM bkpf WHERE ...

" ✅ 正例
SELECT SINGLE amount INTO @DATA(result) FROM bkpf WHERE ...
```

**[L-5] 避免过时的类型转换** *(Major)*
```abap
" ❌ 反例 — 隐式截断风险
MOVE long_string TO short_char20.

" ✅ 正例 — 明确转换意图
short_char20 = CONV char20( long_string ).  " 显式，且有编译期检查
```

---

## 3. Constants（常量）

**[C-1] 用常量替换魔法数字和字面量** *(Major)*
```abap
" ❌ 反例 — 魔法数字/字面量，含义不明
IF status = 'E'.
IF amount > 999999.

" ✅ 正例
CONSTANTS: c_status_error TYPE char1 VALUE 'E'.
CONSTANTS: c_max_amount   TYPE dmbtr VALUE 999999.
IF status = c_status_error.
```

**[C-2] 用枚举类管理相关常量** *(Minor)*
```abap
" ✅ 推荐模式
CLASS order_status DEFINITION ABSTRACT FINAL.
  PUBLIC SECTION.
    CONSTANTS:
      open      TYPE char1 VALUE 'O',
      confirmed TYPE char1 VALUE 'C',
      cancelled TYPE char1 VALUE 'X'.
ENDCLASS.
```

**[C-3] 不允许硬编码业务配置值** *(Major → 在多数企业场景中应为 HIGH)*
```abap
" ❌ 反例 — 硬编码公司代码、工厂、客户端
IF bukrs = '1000'.
IF werks = 'SH01'.
DATA(mandt) = '100'.

" ✅ 正例 — 通过参数、配置表或 SY-MANDT 获取
IF bukrs = iv_bukrs.
IF werks IN s_werks.
DATA(mandt) = sy-mandt.
```

---

## 4. Variables（变量）

**[V-1] 在最小作用域声明变量** *(Minor)*
```abap
" ❌ 反例 — 在方法顶部声明所有变量
METHOD process.
  DATA: name TYPE string.
  DATA: amount TYPE dmbtr.
  DATA: flag TYPE abap_bool.
  " ...100行后才用到 flag

" ✅ 正例 — 在使用处附近声明
METHOD process.
  DATA(name) = get_name( ).
  DATA(amount) = calculate( name ).
```

**[V-2] 不重用变量于不同目的** *(Major)*
```abap
" ❌ 反例 — result 先用于行数，再用于金额
DATA: result TYPE i.
result = lines( orders ).
" ...
result = calculate_amount( ).  " 语义完全不同
```

---

## 5. Tables（内表操作）

**[T-1] 选择合适的内表类型** *(Major)*

| 使用场景 | 推荐类型 | 原因 |
|---------|---------|------|
| 顺序处理、排序后遍历 | `STANDARD TABLE` | 插入快 |
| 频繁按 key 读取（READ TABLE） | `SORTED TABLE` | O(log n) |
| 大量随机 key 查找 | `HASHED TABLE` | O(1) |

```abap
" ❌ 反例 — 对 STANDARD TABLE 使用 WITH KEY 在循环中读取
LOOP AT orders.
  READ TABLE details WITH KEY order_id = orders-id INTO DATA(detail).
ENDLOOP.  " O(n²) 复杂度

" ✅ 正例 — 改用 HASHED TABLE 或 SORTED TABLE
DATA details TYPE HASHED TABLE OF detail_s WITH UNIQUE KEY order_id.
```

**[T-2] 避免在循环内 SELECT** *(Critical)*
```abap
" ❌ 反例 — 经典 1+N 查询，生产环境致命
LOOP AT orders INTO DATA(order).
  SELECT SINGLE * FROM vbap INTO DATA(item)
    WHERE vbeln = order-vbeln.  " 每条 order 都触发一次 DB 调用
ENDLOOP.

" ✅ 正例 — 先 SELECT ... FOR ALL ENTRIES，再用内表关联
SELECT * FROM vbap INTO TABLE @DATA(items)
  FOR ALL ENTRIES IN @orders
  WHERE vbeln = @orders-vbeln.
```

**[T-3] 使用投影而非 SELECT \*** *(Major)*
```abap
" ❌ 反例
SELECT * FROM bkpf INTO TABLE @DATA(docs).

" ✅ 正例
SELECT bukrs, belnr, gjahr, bldat
  FROM bkpf INTO TABLE @DATA(docs)
  WHERE bukrs = @bukrs.
```

---

## 6. Strings（字符串）

**[S-1] 使用模板字面量拼接字符串** *(Minor)*
```abap
" ❌ 反例
CONCATENATE first_name ' ' last_name INTO full_name.

" ✅ 正例
DATA(full_name) = |{ first_name } { last_name }|.
```

**[S-2] 字符串比较注意尾部空格** *(Major)*
```abap
" ❌ 容易出错 — CHAR 类型有尾部空格
DATA: name TYPE char20 VALUE 'SAP'.
IF name = 'SAP'.  " 等价于 'SAP                 '

" ✅ 使用 STRING 类型或显式 CONDENSE
DATA: name TYPE string VALUE 'SAP'.
```

---

## 7. Booleans（布尔值）

**[B-1] 使用 ABAP_BOOL 而非数字标志** *(Major)*
```abap
" ❌ 反例
DATA: is_valid TYPE i.
is_valid = 1.
IF is_valid = 1.

" ✅ 正例
DATA: is_valid TYPE abap_bool.
is_valid = abap_true.
IF is_valid = abap_true.
```

**[B-2] 使用谓词方法表达布尔意图** *(Minor)*
```abap
" ❌ 反例
IF status = 'E' OR status = 'X' OR status = 'A'.

" ✅ 正例
IF is_error_status( status ).
```

---

## 8. Conditions（条件判断）

**[CO-1] 不使用否定条件** *(Minor)*
```abap
" ❌ 反例 — 双重否定难以理解
IF NOT is_not_valid.

" ✅ 正例
IF is_valid.
```

**[CO-2] 提取复杂条件为方法** *(Major)*
```abap
" ❌ 反例 — 多条件难以理解
IF status = 'E' AND amount > 0 AND bukrs = '1000' AND NOT blocked.

" ✅ 正例
IF is_eligible_for_posting( status = status amount = amount ).
```

---

## 9. Methods（方法设计）

**[M-1] 方法应只做一件事** *(Critical)*
```abap
" ❌ 反例 — 方法名暗示它做了多件事
METHOD validate_and_save_and_notify.

" ✅ 正例 — 分离关注点
METHOD validate_order.
METHOD save_order.
METHOD notify_stakeholders.
```

**[M-2] 方法应短小（理想 3-5 语句，上限 20 行）** *(Major)*

*官方依据: "Keep methods short. If a method is more than 20 statements, think of splitting it."*

**[M-3] 最多 3 个 IMPORTING 参数** *(Major)*
```abap
" ❌ 反例
METHODS calculate
  IMPORTING bukrs TYPE bukrs
            belnr TYPE belnr_d
            gjahr TYPE gjahr
            koart TYPE koart
            waers TYPE waers.

" ✅ 正例 — 使用结构封装
METHODS calculate
  IMPORTING document TYPE document_key_s.
```

**[M-4] 用 RETURNING 而非 EXPORTING** *(Minor)*
```abap
" ❌ 反例
METHODS get_total
  EXPORTING ev_total TYPE dmbtr.

" ✅ 正例
METHODS get_total
  RETURNING VALUE(result) TYPE dmbtr.
```

**[M-5] 避免 EXPORTING 和 RETURNING 并用** *(Major)*

---

## 10. Error Handling（错误处理）

**[E-1] 使用类异常而非经典异常** *(Major)*
```abap
" ❌ 反例 — 经典异常（过时）
CALL FUNCTION 'Z_FM'
  EXCEPTIONS
    not_found = 1
    OTHERS    = 2.

" ✅ 正例 — 类异常
TRY.
  z_object->execute( ).
CATCH cx_not_found INTO DATA(ex).
  " 处理异常
ENDTRY.
```

**[E-2] CALL FUNCTION 后必须检查 SY-SUBRC** *(Critical)*
```abap
" ❌ 反例 — 忽略返回码
CALL FUNCTION 'Z_PROCESS_ORDER'
  EXPORTING iv_order = lv_order.
" 直接继续，不知道是否成功

" ✅ 正例
CALL FUNCTION 'Z_PROCESS_ORDER'
  EXPORTING iv_order   = lv_order
  EXCEPTIONS not_found = 1
             OTHERS    = 2.
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE cx_processing_failed.
ENDIF.
```

**[E-3] 不要忽略异常** *(Critical)*
```abap
" ❌ 反例 — 吞掉异常
TRY.
  risky_operation( ).
CATCH cx_root.            " 捕获所有并忽略
ENDTRY.

" ✅ 正例
TRY.
  risky_operation( ).
CATCH cx_specific_error INTO DATA(ex).
  log_error( ex ).
  RAISE EXCEPTION TYPE cx_outer_error
    EXPORTING previous = ex.
ENDTRY.
```

**[E-4] 使用 cx_static_check 还是 cx_dynamic_check？** *(Major)*

| 类型 | 何时使用 |
|------|---------|
| `CX_STATIC_CHECK` | 调用者**可以合理处理**的可预期错误（如文件不存在） |
| `CX_DYNAMIC_CHECK` | 编程错误（如空指针），调用者无法预先避免 |
| `CX_NO_CHECK` | 严重系统错误，不应在业务代码中处理 |

---

## 11. Comments（注释）

**[CM-1] 用代码表达意图，而非注释** *(Major)*
```abap
" ❌ 反例 — 注释解释代码在做什么（代码本身应该说清楚）
" Check if the order is valid
IF status = 'C' AND amount > 0.

" ✅ 正例 — 代码自解释
IF is_confirmed_order( status ) AND has_amount( amount ).
```

**[CM-2] 注释解释"为什么"，不解释"是什么"** *(Minor)*
```abap
" ✅ 好注释 — 解释业务原因
" SAP Note 2847563: FI posting requires BKPF lock before BSEG update
" to prevent duplicate document numbers in concurrent sessions
CALL FUNCTION 'ENQUEUE_EFIBL'.
```

**[CM-3] 禁止注释掉的代码进入生产** *(Major)*
```abap
" ❌ 反例 — 注释掉的旧代码
* DATA: old_var TYPE string.
* PERFORM old_routine.
```

---

## 12. Formatting（格式化）

**[F-1] 每行只写一条语句** *(Major)*
```abap
" ❌ 反例
DATA: a TYPE i. DATA: b TYPE i.

" ✅ 正例
DATA: a TYPE i.
DATA: b TYPE i.
```

**[F-2] 使用 Pretty Printer 统一格式** *(Minor)*

SAP 官方建议每次保存代码时运行 Pretty Printer（SE38/SE80 中可配置）。

---

## 13. Testing（测试）

**[TS-1] 编写 ABAP Unit Tests** *(Major)*

*官方依据: "Untested code is broken code. Write unit tests."*

关键检查点：
- 是否有对应的测试类 (`*_TEST` 或独立测试 include)
- 关键业务逻辑是否有测试覆盖
- 测试是否使用 Test Double / Mock 替代真实 DB 调用

**[TS-2] 测试代码与生产代码分离** *(Minor)*
```abap
" ✅ 正确方式 — 使用 FOR TESTING 标记
CLASS order_validator_test DEFINITION
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.
```

---

## Reference Links

| 资源 | URL |
|------|-----|
| Clean ABAP Style Guide（官方完整版） | `github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md` |
| ABAP Code Review Guideline | `github.com/SAP/styleguides/blob/main/abap-code-review/ABAPCodeReview.md` |
| code-pal-for-ABAP（自动化检查工具） | `github.com/SAP/code-pal-for-abap` |
| ABAP Cleaner（100+ 自动修复规则） | `github.com/SAP/abap-cleaner` |

---
*本文件为 SAP 官方 Clean ABAP Style Guide 的审查导向摘录，供 AI Agent 在 [STD] 维度引用。*
*原始规范以 CC BY 4.0 许可开放，版权归 SAP SE 及贡献者所有。*
