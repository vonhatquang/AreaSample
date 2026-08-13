# Data type conversion — discussion note (design only)

**Status:** Discussion / design. **Do not implement** until policy is agreed.  
**Audience:** Quang (internal), then selected points for JP customer discussion.  
**Context:** Feedback on the procedure (手順書) is accepted. Customer added a missed point: no measure when a data type is **not in the conversion rules**. Tables vs views, and CDC/ALTER constraints, are the reason this matters.

---

## 1. Customer email (source)

Customer (JP) to Vo, Nhat Quang:

- Procedure updates from prior review: OK.
- One additional point they missed:
  - There is **no measure** when a data type **not in the conversion rules** appears.
  - Data types for **tables** need careful design.
  - **Views** do not specify types in CREATE; Snowflake assigns them. Secondary use is almost only end users.
  - **Tables** are used by system processing. Wrong definitions waste tests and cause errors.
  - Current state is structured Raw. Once it becomes Raw + continuous CDC, **ALTER cannot be done casually**.
- Customer’s personal limit / proposed steps:
  1. Confirm Qlik conversion rules at document level.
  2. On 保全 (preservation/staging), compare SQL Server DDL vs Snowflake DDL; confirm no contradiction with conversion rules.
  3. Create a conversion table (変換表) and apply it.
- They are **not** asking the Vietnam team to do everything. They want Quang’s opinion.

---

## 2. Direct answer (what to say in the meeting)

**①②③ are necessary, but not sufficient.**

They cover **design-time mapping** for types we already know. They do **not** cover:

- Types that appear later (new table, new column, type change).
- Types that exist in source but not in the 保全 sample.
- What the pipeline must **do** when an unmapped type is seen (stop vs fallback).
- Who **approves** an exception.
- How to change schema after CDC go-live without casual ALTER.

Customer’s instinct is correct:

- **Table = strict** (system, tests, CDC).
- **View = looser** (inferred types, user 2次利用).
- **Raw + CDC = treat type as a contract**, not something to fix later with ALTER.

Quang does not need to be a Qlik expert. Quang needs a **decision framework**: known type vs unknown type, table vs view, design-time vs run-time.

---

## 3. Assessment of customer steps ①②③

| Step | What it covers | What it does not cover |
|---|---|---|
| ① Qlik conversion rules (docs) | Default Qlik → Snowflake mapping for documented types | Task-level custom mapping; product/version differences (Replicate vs Cloud Data Integration); types SQL Server has but Qlik docs treat poorly; “loads successfully” ≠ “semantically correct” |
| ② Compare SQL Server DDL vs Snowflake DDL on 保全 | Types **currently present** in that environment; contradictions with the rule doc | Types not yet used; other DBs/schemas out of that env; UDT/alias/computed; real precision/timezone/null behavior; future CDC drift |
| ③ 変換表 | Lookup during table design; shared language between JP and VN | Unknown type at run-time; schema drift after go-live; approval path; test/acceptance of semantic correctness |

### Risk if we only do ①②③

- First unmapped type (`DATETIMEOFFSET`, `SQL_VARIANT`, `GEOGRAPHY`, UDT, …) → job fails **or** silently wrong column type.
- Tests run on wrong types → wasted 試験, late production errors.
- After Raw CDC starts, fixing a type may mean recreate + backfill, not `ALTER COLUMN`.

**Verdict:** Keep ①②③ as the baseline. Add inventory, exception policy, and CDC change management. That is the actual answer to 「この点どうしましょうか」.

---

## 4. Table vs View (agree with customer, make it a rule)

### Tables

- Used by system processing.
- Types are part of the **interface** for jobs, tests, downstream apps.
- Wrong type → failed casts, precision loss, timezone bugs, CDC apply errors.
- Once CDC is continuous on Raw, changing a column type is expensive and risky.
- **Policy: explicit mapping only. No inferred types. No silent fallback.**

### Views

- CREATE often does not declare types; engine infers from expressions.
- Secondary use is mostly human / ad-hoc.
- Wrong inference is annoying, usually not a system outage.
- **Policy: may allow inferred types and a documented fallback (e.g. VARCHAR/VARIANT) with warning.**
- Still record inferred types in a catalog so 2次利用 users are not surprised.

### Practical split

| Object | Type source | Unmapped type | ALTER later |
|---|---|---|---|
| TABLE (system / Raw CDC) | 変換表 only | **STOP** | Avoid type change; add column or rebuild |
| VIEW (user) | Inference OK | Fallback + warning OK | Recreate view is cheap |

Do not spend equal design energy on views and tables.

---

## 5. Why CDC Raw makes this urgent

While data is “structured Raw” in a loadable/rebuildable state, a wrong type can still be fixed by recreate.

After **continuous CDC**:

- Apply process expects a stable target schema.
- `ALTER TYPE` on Snowflake is limited; some changes are not in-place.
- Narrowing type, changing TIMESTAMP_NTZ ↔ TIMESTAMP_TZ, changing NUMBER precision, changing NULL → NOT NULL are **breaking**.
- Widening VARCHAR length is often possible; still needs a change process, not ad-hoc ALTER.
- Adding a column is usually safer than changing an existing column (prefer append).
- Historical data already landed may not match the new type.
- Source CDC itself has limits (LOB/`varchar(max)`/`XML` capture behavior on SQL Server).

**Design rule:** Treat target column type as a **long-lived contract**. Decide before first CDC cutover. After cutover, schema change = change request, not a DBA tweak.

---

## 6. Cases ①②③ do not automatically catch

Group into four buckets for discussion (do not dump every type name on the customer unless asked).

### 6.1 Type maps, but meaning is wrong

- `DATETIMEOFFSET` → `TIMESTAMP_NTZ` (timezone dropped).
- `DATETIME2(7)` fractional seconds truncated.
- `MONEY` / `SMALLMONEY` scale/rounding.
- `FLOAT` / `REAL` vs `NUMBER` (binary vs decimal).
- `NVARCHAR(MAX)` / `VARCHAR(MAX)` length and CDC capture limits.
- `BIT` → `BOOLEAN` vs `NUMBER(1)` (downstream SQL compatibility).
- `UNIQUEIDENTIFIER` → `STRING` vs `BINARY` (join/filter behavior).
- Collation / Japanese supplementary characters.

### 6.2 SQL Server-specific / poorly mapped types

- `TIMESTAMP` / `ROWVERSION` (binary, **not** datetime).
- `XML`, `SQL_VARIANT`.
- `GEOGRAPHY`, `GEOMETRY`, `HIERARCHYID`.
- User-defined types (UDT) and alias types (`sys.types` where `is_user_defined = 1`).
- Legacy `TEXT` / `NTEXT` / `IMAGE`.
- `JSON` (if used), newer types.

### 6.3 Not a data type, but still breaks tables

- `IDENTITY` (generation stays on source; target is just a number).
- Computed columns (persist vs not; CDC may not ship them as expected).
- `NULL` / `NOT NULL`, defaults.
- `SPARSE`, `FILESTREAM`.
- Collation differences.
- Column order (some CDC/apply tools care about ordinal).

### 6.4 After go-live (schema drift)

- New column on source.
- `VARCHAR(50)` → `VARCHAR(200)`.
- Nullability change.
- Precision/scale change.
- Type change (`INT` → `BIGINT`, `DATETIME` → `DATETIME2`).
- New table in scope.
- Rename / drop column.

①②③ on 保全 **today** cannot certify ⑥.

---

## 7. Recommended process (keep ①②③, add A–E)

This is the proposed “limit” that is actually safe enough. Still **design**, not implementation.

```
① Qlik mapping docs (version-pinned)
② DDL compare on 保全 (SQL Server vs Snowflake)
③ 変換表 (living document)
── added ──
A. Inventory of types actually used on source (sys.columns)
B. Exception policy: TABLE=STOP, VIEW=fallback (or customer-chosen variant)
C. CDC schema-change process (no casual ALTER of existing types)
D. Semantic acceptance (sample data, not DDL-only)
E. Periodic re-scan after go-live (drift detection)
```

### A. Inventory (source of truth = SQL Server, not memory)

Purpose: 変換表 must cover **used** types, not only types someone remembered.

Suggested inventory (run later, not now):

- Distinct `system_type` / `user_type` + count of columns.
- Max length, precision, scale, nullability observed.
- Which tables/columns use “yellow/red” types.
- UDT/alias → base type.
- List of schemas in CDC scope (保全 must include all of them, or ② is only a sample).

If 保全 does not contain every in-scope schema, say so explicitly: **② is a sample check, not a full proof.**

### B. Exception policy (this is the missing 措置)

Need **one operational rule**, written in the 手順書.

**Recommended default (propose to customer):**

| Situation | TABLE (system / Raw CDC) | VIEW (user) |
|---|---|---|
| Type in 変換表 | Use mapped type | Inference or mapped type |
| Type not in 変換表 | **Do not guess. Stop / fail the task. Open ticket. No table create/alter until approved.** | Fallback `VARCHAR` or `VARIANT` + warning log |
| Mapping exists but confidence = Yellow | JP approve before create | Warning OK |
| Mapping confidence = Red | Stop; need design | Stop or fallback per JP |

**Why STOP on tables:** Silent fallback (`VARCHAR`/`VARIANT` everywhere) makes tests pass and production semantics fail. Customer already said wrong table types waste 試験.

Alternative the customer might prefer (ask, do not assume):

- **Strict:** STOP (recommended for CDC Raw tables).
- **Lenient:** Cast to `VARCHAR`/`VARIANT`, alert, continue (acceptable for sandbox/views only).
- **Hybrid:** STOP in 本番/保全-CDC; lenient in a landing/debug area if they ever have one.

**Do not mix:** a system table must not be “sometimes VARCHAR because Qlik didn’t know the type.”

### C. CDC change management

Write as rules, not as “be careful”:

1. **Forbidden without change request:** change existing column type, shrink length, change timezone type, change nullability to stricter.
2. **Usually allowed with process:** add nullable column at end; widen VARCHAR (confirm Qlik + Snowflake + apply).
3. **Breaking:** new type for old column → new column or new table + backfill; dual-read period if needed.
4. **Source freeze (optional but valuable):** short DDL freeze on source before cutover, plus a final inventory diff.
5. **Detection:** Qlik alert and/or scheduled compare of `sys.columns` vs 変換表 / Snowflake `INFORMATION_SCHEMA`.
6. **SLA:** who is notified, how fast to stop apply, who approves the new mapping.

### D. Semantic acceptance (DDL match is not enough)

After mapping, for a **pilot set of tables** (not all tables on day one):

- Row counts source vs target.
- NULL ratios.
- Min/max for numeric and datetime.
- Timezone-sensitive columns: same instant?
- Money/decimal: scale preserved?
- Binary/GUID: round-trip?
- Japanese text: no `?` replacement, supplementary chars.

This prevents “DDL looks mapped, data is wrong.”

### E. Living 変換表 + re-scan

変換表 is not a one-off Excel. After go-live:

- Re-run inventory periodically.
- Diff vs 変換表.
- New type → same exception path as B.

---

## 8. Confidence model for the 変換表 (design of the table, not the data)

Do not only store `SQL Server type → Snowflake type`. Store **how sure we are**.

| Confidence | Meaning | Action |
|---|---|---|
| Green | Documented, common, no known semantic loss | Auto-apply from 変換表 |
| Yellow | Maps, but precision/timezone/length risk | JP review on first use per type (not per column, unless column is special) |
| Red | No safe default (XML, UDT, GEOGRAPHY, SQL_VARIANT, …) | Stop; design per type or per column |

Suggested columns for 変換表 (document template only):

| Field | Purpose |
|---|---|
| sqlserver_type | e.g. `datetimeoffset`, `nvarchar` |
| length_precision_scale_rule | How to map `(p,s)` / `max` |
| snowflake_type | e.g. `TIMESTAMP_TZ(9)`, `VARCHAR` |
| nullability_rule | Usually preserve |
| qlik_doc_mapping | What Qlik says (version + link/section) |
| qlik_actual_on_保全 | What ② observed |
| semantic_risk | timezone / rounding / binary / collation |
| confidence | Green / Yellow / Red |
| allowed_on_table | Y/N |
| fallback_if_any | only for views, or none |
| owner_approval | who signed |
| notes | CDC limitations, IDENTITY, etc. |

If Qlik 保全 output **disagrees** with Qlik docs → 保全 wins as “actual,” docs get a note, JP confirms which target type they want.

---

## 9. Ideas on architecture (only if customer wants options)

Customer asked “どうしましょうか”, not “redesign the warehouse.” Offer these only as options; **default is: typed tables + STOP + 変換表.**

### Option 1 — Strict typed Raw (likely their current direction)

- Target tables fully typed from 変換表.
- Unknown type stops the pipeline.
- Highest safety for system use.
- Highest need for inventory + exception process.

### Option 2 — Landing as semi-structured, typed layer downstream

- Raw as `VARIANT`/`VARCHAR` (or Qlik landing), curated tables typed.
- ALTER of Raw types becomes less critical.
- Extra storage, extra jobs, extra tests.
- May conflict with “structured Raw” already in flight.
- **Do not propose as default** unless they cannot freeze types.

### Option 3 — Hybrid

- High-value system tables: Option 1.
- Wide/unknown/user dumps: Option 2 or views.
- Matches customer’s table vs view intuition.

Recommendation to Quang: **stay on Option 1 + policy B/C**, unless customer explicitly wants a VARIANT landing zone.

---

## 10. RACI (so VN is not “doing everything”)

Customer said they do not want VN to own the whole problem. Align with that.

| Work | JP customer | VN team | Qlik/infra (if separate) |
|---|---|---|---|
| Pin Qlik product + version + mapping doc | A | C | R |
| Inventory `sys.columns` in scope | A (scope) | R | C |
| ② DDL compare on 保全 | A | R | C |
| Draft 変換表 | C | R | C |
| Approve 変換表 + Yellow/Red types | **A/R** | C | I |
| Exception policy (STOP vs fallback) | **A/R** | C | I |
| CDC change process + SLA | **A/R** | C | R (alerts) |
| Implement tables per approved 変換表 | I | R | C |
| Pilot semantic checks | A (pass/fail) | R | C |

**R** = does, **A** = accountable, **C** = consult, **I** = inform.

VN can do inventory, compare, draft mapping. VN should **not** silently decide timezone, money scale, or unmapped types.

---

## 11. Questions Quang should ask (looks experienced; no tool mastery required)

1. **Scope:** Only current structured Raw tables, or every SQL Server table that may enter CDC later?
2. **Qlik product:** Replicate vs Cloud Data Integration? Version? Custom type mapping in the task, or defaults only?
3. **Unmapped type:** Prefer **STOP** or **fallback** for tables? (Propose STOP.)
4. **Approver:** Who signs Yellow/Red mappings — customer, or VN proposes and customer stamps?
5. **保全 coverage:** Does 保全 include **all** in-scope schemas? If not, ② is a sample.
6. **After go-live:** Who detects source DDL change? Qlik alert vs periodic `sys.columns` diff? Stop apply or keep going?
7. **SLA:** How long can CDC apply stay stopped while mapping is decided?
8. **Acceptance:** Is DDL match enough, or do we need sample semantic checks on a pilot?
9. **Timezone standard:** `TIMESTAMP_NTZ` vs `TIMESTAMP_TZ` vs `TIMESTAMP_LTZ` for datetimeoffset/datetime2?
10. **BIT / GUID / MONEY:** any downstream system that requires a specific Snowflake type?
11. **LOB/XML:** are those columns in CDC scope at all? (If yes, Red.)
12. **Source DDL freeze** before cutover: possible?

Questions **3 and 6** are the ones ①②③ never answer. Those are the meeting goal.

---

## 12. How Quang should stand in the discussion (no fake expertise)

**Do**

- Agree that tables need a hard contract; views can be looser.
- Agree ①②③ are the right design-time core.
- Add: exception 措置 + CDC change process + inventory (because ② cannot see the future).
- Offer VN work: inventory, DDL compare, draft 変換表.
- Leave policy and Yellow/Red types to JP.
- If asked “how does Qlik map X?”: “We confirm on the pinned doc and on 保全, then write it in 変換表. We do not assume.”

**Do not**

- Pretend to know every Qlik default by heart.
- Accept “VN decides all types.”
- Accept silent VARCHAR fallback on system tables “to keep the job green.”
- Promise that ② on 保全 proves all future types.
- Open with a 20-type lecture. Use the four buckets if they ask “what could go wrong?”

**One sentence for the meeting**

> ①②③ は設計時の変換表には十分です。不足しているのは、変換表に無い型の停止ルールと、CDC開始後の型変更をALTERしない変更管理です。テーブルは停止、Viewはフォールバック可、を提案します。

---

## 13. Suggested meeting agenda (30–40 min)

1. Confirm shared goal: no wasted 試験 from wrong table types; no casual ALTER after CDC.
2. Confirm ①②③ as design-time baseline.
3. Decide exception 措置: TABLE STOP vs fallback (need a decision, not “検討”).
4. Decide 保全 = full scope or sample.
5. Decide who approves Yellow/Red.
6. Decide drift detection after go-live (even a lightweight rule).
7. Agree outputs (next section) and who drafts vs who signs.
8. Optional: timezone / MONEY / MAX types if they have known critical columns.

---

## 14. Documents to produce later (still not implementing now)

When they say go:

1. **変換表** (template in §8) — living.
2. **未定義型ランブック** — detect → stop → ticket → approve → apply → restart.
3. **Type inventory** from SQL Server (in-scope).
4. **② compare result** — contradictions list (doc vs 保全 vs 変換表).
5. **CDC schema-change rules** — allowed / forbidden / rebuild.
6. **Pilot acceptance checklist** — semantic checks.
7. **RACI** — one page.
8. **手順書 delta** — short section “変換ルール外のデータ型” + “CDC後のDDL変更”.

Until then, this file is the internal idea dump.

---

## 15. 手順書 — suggested text (JP, for later paste)

Use after customer agrees. Draft only.

### 15.1 変換ルール外のデータ型

- テーブル（システム利用・CDC対象）は、変換表に定義のないデータ型を推測して作成・変更しない。
- 該当を検知した場合は処理を停止し、変換表の更新と承認後に再開する。
- View（ユーザ2次利用）は、Snowflakeの型推論を許容する。変換表外の場合は VARCHAR または VARIANT へのフォールバックと警告を許容する（方針確定後に一方を記載）。

### 15.2 変換表

- Qlik公式の変換規則（製品・バージョンを明記）を正とする。
- 保全環境で SQL Server DDL と Snowflake DDL を突合し、公式規則との差分を変換表に記録する。
- 公式と実測が異なる場合は実測を「事実」として記録し、採用型は承認者が決める。

### 15.3 CDC開始後の型変更

- 既存列の型変更、桁縮小、NOT NULL強化は原則禁止（変更管理＋再作成/バックフィル）。
- 列追加（NULL可）および桁拡大は、Qlik・Snowflake・適用プロセスの確認後に実施可能とする。
- ソースDDL変更の検知方法と停止判断は別途運用で定める。

---

## 16. Draft reply to customer (JP)

Quang can send as-is or shorten.

> ご相談ありがとうございます。ご指摘の通り、変換ルール外のデータ型への措置は必要だと思います。
>
> ①〜③の手順自体は賛成です。設計時点の変換表を作るにはこの3点が妥当だと思います。
> 一方、保全環境のDDL比較は「今存在している型」の確認には有効ですが、今後CDCで増える型や、保全に含まれないスキーマまでは保証できません。また、Qlikのドキュメント上の変換と、実際のタスク出力が一致するとも限りません。Raw＋継続CDCでは安易なALTERが難しいため、設計時の変換表に加えて、例外時の運用ルールが必要だと考えます。
>
> 私の案は以下です。
>
> - テーブル（システム利用・CDC）は、変換表に無い型を推測せず、処理を停止し、確認・承認後に定義する
> - View（ユーザ2次利用）は推論型でリスクが低いため、警告付きのフォールバック（VARCHAR / VARIANT）を許容してもよい
> - 事前にSQL Serverで利用中のデータ型を棚卸しし、変換表の対象漏れを防ぐ（保全が対象全スキーマを含むかも確認）
> - 変換表には「公式マッピング / 保全の実測 / 採用型 / 確度（問題なし・要確認・停止）」を残す
> - 稼働後の型変更・桁縮小は通常ALTERではなく、変更管理（新規列追加または再作成）とする
>
> ベトナム側で全部を判断する、というより、棚卸し・DDL突合・変換表の下書きは進められます。  
> 未定義型の扱い（停止 or 代替型）と、例外の承認者だけ、方針としてご判断いただきたいです。
>
> 必要なら、上記を手順書の追記案として短く落とします。

Shorter spoken version:

> ①②③は賛成です。足りないのは「変換表に無い型のとき止めるか」と「CDC後は型を安易にALTERしない」の2点だと思います。テーブルは停止、Viewはフォールバック、を提案します。

---

## 17. What “cover every case” actually means

Customer and Quang both want coverage. **Every case cannot be a pre-listed type.** Coverage means:

1. **Known types** → 変換表 (①②③ + inventory A).
2. **Unknown types** → exception path B (STOP/fallback + owner).
3. **Type changes later** → process C, not hero ALTER.
4. **Mapped but semantically wrong** → pilot checks D + Yellow flag.
5. **Out of inventory scope** → explicit “not in scope” (do not pretend 保全 proved it).

If those five are written, the 手順書 covers the cases that matter. Listing 40 SQL Server types in an appendix is optional; the **path for the 41st type** is mandatory.

---

## 18. Internal checklist for Quang (discussion day)

- [ ] Do not volunteer VN to own type decisions.
- [ ] Get a yes/no on TABLE=STOP.
- [ ] Get a name for the approver of exceptions.
- [ ] Confirm 保全 = full scope or sample.
- [ ] Confirm Qlik product/version for ①.
- [ ] Offer to draft 変換表 + 手順書追記 after they choose STOP vs fallback.
- [ ] Do not start implementing jobs, DDL generators, or inventory scripts until they agree.

---

## 19. Out of scope for this note

- Implementing Qlik tasks, Snowflake DDL, or SQL inventory scripts.
- Filling the actual 変換表 with mappings (needs their Qlik version + 保全 DDL).
- Choosing final Snowflake types for DATETIMEOFFSET / MONEY / MAX without customer input.

---

## 20. One-page summary

| Topic | Position |
|---|---|
| Customer ①②③ | Keep |
| Enough alone? | No |
| Missing 措置 | Unmapped type on **tables** → stop + approve |
| Views | Inference / fallback OK |
| 保全 DDL compare | Necessary; proves present types only |
| Inventory | Required so 変換表 is not based on memory |
| CDC | No casual ALTER of existing types |
| VN role | Draft and compare; not approve Yellow/Red |
| JP role | Policy + approvals |
| Next artifact after agreement | 手順書追記 + 変換表 template |

---

*Internal discussion note. Not a project commitment until JP agrees.*
