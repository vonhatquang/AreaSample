# 返信案：データ型変換手順（お客様向け）

Quang の下書きを整えた版。送る前のチェック用。

---

## 評価（内部）

| 点 | 判定 |
|---|---|
| 変換の正を Qlik にする | 良い |
| Matrix（SQL Server→Qlik→Snowflake）を作る | 良い |
| その後に SQL Server→Snowflake の Mapping を作る | 良い |
| テーブル定義を一時停止してから型を固める | 良い（再作業防止） |
| ① 現行DDLの型一覧 | 良い（ルール外を見つける前提） |
| ②「Qlik内部型だけ」一覧 | やや弱い。③のリンクで足りる。①と突合する旨を足す |
| ルール外の型の措置 | **不足**（お客様が最初に聞いた点） |
| 保全の実測DDL確認（お客様②） | **不足** |
| ツール反映（⑤） | 方針合意・Mapping承認の**後**と書く |

結論: 方向は正しい。送ってよいが、下の追記を入れた方がお客様の懸念に答えられる。

---

## 送信用（推奨）

お疲れ様です。

データ型の変換について、以下の認識・手順で進めたくご確認をお願いいたします。

■ データ型の変換手順（案）

① 現行システムの DDL（CREATE TABLE）を読み、全テーブルのフィールドのデータ型を取得し、使用中データ型の一覧を出力する。

② Qlik Replicate の公式ドキュメントから、変換規則を確認する。  
　・Qlik 内部型  
　　https://help.qlik.com/en-US/replicate/May2026/Content/Replicate/Main/Endpoints/att_rep_data_types.htm  
　・SQL Server 型 → Qlik 内部型  
　　https://help.qlik.com/en-US/replicate/May2026/Content/Replicate/Main/SQL%20Server/SQLServerDB_source_DataTypes.htm  
　・Qlik 内部型 → Snowflake 型  
　　https://help.qlik.com/en-US/replicate/May2026/Content/Replicate/Main/Snowflake-Target/Snowflake-target-datatypes.htm  

③ 「SQL Server 型 → Qlik 内部型 → Snowflake 型」の Matrix を作成する。  
　あわせて、①の使用中型と突合し、次のように分類する。  
　・両方にある型 … Matrix に載せて変換する  
　・現行DDLにあるが Qlik に無い／非サポートの型 … **ルール外**（推測して仮の型を付けない）  
　・Qlik にあるが現行DDLに無い型 … 参考（今は未使用）

④ ③に基づき、「SQL Server 型 → Snowflake 型」の Mapping 一覧を作成する。  
　※ テーブル定義に使うのは Snowflake 型。Qlik 内部型は変換の根拠として Matrix に残す。  
　※ 変換の正は Qlik とする（Snowflake 公式の別マッピングで置き換えない）。

⑤ ルール外の型が出た場合は、当該テーブルの定義を止め、ご確認・承認後に Mapping を更新してから作成する。  
　（VARCHAR 等への仮置きは、CDC 開始後に直せなくなるため行わない）

⑥ 可能であれば保全環境で、SQL Server DDL と Snowflake 実測 DDL を比較し、④の Mapping と矛盾がないことを確認する。  
　（以前ご提示の②に相当。公式と実測が異なる場合は実測を事実として記録し、採用はご判断いただきたいです）

⑦ Mapping 一覧の方針・内容をご確認いただいたうえで、自動生成ツール／レビューツールへ反映する。

■ テーブル定義作成の予定

① 現在のテーブル定義作成作業は一時的に停止する。  
② 上記のデータ型変換（Mapping 確定）を先行する。  
③ Mapping の結果に基づき、テーブル定義作成を再開する。

認識に相違・不足がございましたらご指摘ください。  
特に、ルール外の型を「作成停止」とする点と、保全での実測確認を残す点について、ご意見を伺えますと幸いです。

よろしくお願いいたします。

---

## 元案との差分（送る理由）

1. **①と Matrix の突合・分類**を明示 → お客様の「ルールに無い型の措置が無い」に直接答える。  
2. **⑤ ルール外は停止**を手順に入れる → 仮置きしない理由（CDC後に直せない）も一文。  
3. **⑥ 保全**を残す → お客様が最初に出していた②。落とすと認識ずれ。  
4. **ツール反映は Mapping 確認後** → 未承認の Mapping をツールに入れると再作業。  
5. ②を「内部型だけ一覧」から「公式の変換規則確認（3リンク）」に整理 → 作業が分かりやすい。

スケジュール停止→型→再開は、元案のまま採用でよい。
