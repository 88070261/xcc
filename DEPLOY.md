# 部署与配置说明（考试训练系统）

四个文件：`index.html`（学生首页）、`quiz.html`（答题）、`admin.html`（老师统计）、`import.html`（题库导入）、`search.html`（题库搜索）。
纯静态、无构建步骤，可直接双击运行或托管到任意静态服务。

---

## 一、替换 Supabase Key

每个文件顶部都有同一段初始化代码，改这两个常量即可：

```js
const SUPABASE_URL = 'https://qfpdjtjstqpeawmzesrl.supabase.co';
const SUPABASE_ANON_KEY = 'sb_publishable_Rl7i4y-_sQE8J8U1_3eieQ_gfkywzMk';
const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

> 当前用的是你提供的 Anon Key，已内置在四个文件里，可直接用。

---

## 二、必看：Supabase 权限（RLS）配置

我已实测你项目里四张表的真实结构和权限，结论如下：

| 表 | 学生端需要 | 当前状态 |
|---|---|---|
| `questions` | 读（SELECT） | ✅ 已开放 |
| `questions` | 写（导入用 INSERT） | ❌ **被 RLS 拦截**，导入功能必须先开权限 |
| `exam_records` | 写（INSERT） | ✅ 已开放 |
| `wrong_books` | 写（UPSERT） | ✅ 已开放 |
| `user_status` | 写（UPSERT 心跳） | ✅ 已开放 |

**导入功能要能用，请到 Supabase 后台给 `questions` 表开放 INSERT 权限**，二选一：

- 方式 A（最简单，仅适合内部/测试环境）：在 `questions` 表 → **RLS** 里直接关掉 RLS，或加一条 `Enable read/write for all (anon)` 的 INSERT 策略。
- 方式 B（推荐，安全）：在 SQL Editor 执行

```sql
-- 允许匿名角色写入 questions（导入题库用）
create policy "anon insert questions"
  on questions for insert
  to anon
  with check (true);
```

> 提示：`wrong_books` 的「同一人同一题错多次 wrong_count+1」依赖 `(student_name, question_id)` 的唯一约束。
> 如果还没有，执行：
> ```sql
> alter table wrong_books add constraint wrong_books_uniq unique (student_name, question_id);
> ```

### 整合初始化 SQL（在 SQL Editor 一次性执行）

把下面整段粘贴到 Supabase 后台 **SQL Editor** 运行即可，包含：开 `questions` 读取策略（**学生端能练习的前提**）、加 `option_e~i` 列、`questions` 写入策略、`wrong_books` 唯一约束（均已做幂等处理，重复执行不会报错）：

```sql
-- 1) 允许匿名读取 questions（学生端练习/搜索必须，否则 anon 查不到任何题目）
do $$
begin
  if not exists (
    select 1 from pg_policies
    where schemaname='public' and tablename='questions' and policyname='anon select questions'
  ) then
    create policy "anon select questions"
      on questions for select
      to anon
      using (true);
  end if;
end $$;

-- 2) 增加多选项列（支持 4~9 个选项）
alter table questions
  add column if not exists option_e text,
  add column if not exists option_f text,
  add column if not exists option_g text,
  add column if not exists option_h text,
  add column if not exists option_i text;

-- 3) 允许匿名写入 questions（后续用 import.html 前端导入时需要；本次已用 service_role 后端导入，可二选一）
do $$
begin
  if not exists (
    select 1 from pg_policies
    where schemaname='public' and tablename='questions' and policyname='anon insert questions'
  ) then
    create policy "anon insert questions"
      on questions for insert
      to anon
      with check (true);
  end if;
end $$;

-- 4) wrong_books 错题累加所需的唯一约束（Postgres 不支持 ADD CONSTRAINT IF NOT EXISTS，用 DO 块判断）
do $$
begin
  if not exists (
    select 1 from pg_constraint
    where conname='wrong_books_uniq' and conrelid='wrong_books'::regclass
  ) then
    alter table wrong_books add constraint wrong_books_uniq unique (student_name, question_id);
  end if;
end $$;
```

---

## 三、部署到 Cloudflare Pages（3 种方式）

### 方式 1：拖拽上传（最快，无需 Git）
1. 把这四个 `.html` 文件放在同一个文件夹（也可加本 `DEPLOY.md`）。
2. 打开 https://dash.cloudflare.com → **Workers & Pages** → **Create** → **Pages** → **Upload assets**。
3. 拖入该文件夹，点 **Deploy**。
4. 部署完成后会得到一个 `xxx.pages.dev` 域名，直接访问即可。

### 方式 2：关联 Git 仓库（推荐，支持自动更新）
1. 把四个文件推到 GitHub / GitLab 仓库根目录。
2. Cloudflare Pages → **Create** → 选 **Connect to Git** → 授权并选择仓库。
3. 构建设置：**Framework preset = None / 无**，**Build command 留空**，**Build output directory = `/`**（或仓库根目录）。
4. 保存并部署。以后 `git push` 会自动重新部署。

### 方式 3：Cloudflare CLI（wrangler）
```bash
npx wrangler pages deploy . --project-name exam-trainer
```

> 静态站点无需函数（Functions），纯 HTML+JS 即可。自定义域名在 Pages 项目的 **Custom domains** 里绑定。

---

## 四、重要数据结构说明（与你 Excel 的对应关系）

我已按你的表结构（`id, question, option_a~d, answer, chapter, difficulty, explanation`）实现了导入映射：

- `编号` → `id`（整数）
- `题目` → `question`
- `选项`（按换行拆分）→ `option_a`~`option_d`（自动去掉 `A. ` 这类前缀）
- `答案` → `answer`（提取字母；多选题用逗号连接，如 `C,F,H,I`）
- `知识点` → `chapter`
- `难度` → `difficulty`（`简单`→1，`中等`→2，`难/困难`→3，**该列在库里是整型**）
- `解释` 你的 Excel 里没有，留空

### ✅ 多选题 9 个选项的兼容（已支持）

你的 Excel 里 **318 道多选题中有 99 道选项超过 4 个**（最多 9 个，如「介入医学可分为…」有 A~I）。
`import.html` 和 `quiz.html` **已经改为支持 `option_a ~ option_i` 共 9 个选项**，导入时按行拆分全部写入，答题页也会渲染对应数量的选项。

**还需要你做一步：给 `questions` 表加 5 个列**（加列是 DDL，Anon Key 无权限，请在 Supabase 后台 **SQL Editor** 执行）：

```sql
alter table questions
  add column option_e text,
  add column option_f text,
  add column option_g text,
  add column option_h text,
  add column option_i text;
```

> 执行后刷新表结构即可。导入时 `import.html` 会把第 5~9 个选项写进 `option_e~i`，答题页 `quiz.html` 的 SELECT 也已包含这些列。
> 若某题选项 ≤4 个，多余的 `option_e~i` 写入 `null`，不影响展示。

> 提示：加列前若已用旧逻辑导入过数据，建议清空 `questions` 后重新导入一次，保证 `option_e~i` 被正确填充。

---

## 五、使用流程

1. `import.html` → 上传题库 Excel，导入到 `questions`。
2. `index.html` → 学生输入姓名，可组合筛选：**按知识点专项突破**（选章节）、**难度**（简单/中等/困难）、**只练我的错题**（勾选后只抽该生错题，可再叠加章节/难度）。随机抽 10 题，每 30 秒上报一次在线状态。
3. `quiz.html` → 逐题作答，可单题提交看解析，也可统一交卷；错题写入 `wrong_books`。
4. `admin.html` → 密码 `123456` 进入，看在线学生 / 章节成绩 / 高频错题。
5. `search.html` → 输入关键词（如「消防」「牛顿定律」「CT」），跨题目与所有选项模糊搜索；默认隐藏答案，点「显示答案」自测。
