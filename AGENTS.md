# AGENTS.md —— LenoMusic 的默认写法

> 给 AI / 新加入的人。**新代码照这 5 条写**；旧写法见文末"遗留（不要扩散）"。
> 完整论证见 `Leno/docs/idioms.md`。

| 场景 | 用这个 | 不要 |
|---|---|---|
| 数据 | `struct` + `Array[struct]` | `Array[Array[string]]`（本站有 **189 处**，全是历史包袱）|
| 状态 | `enum { todo = 0 ... }` | 字符串码 `"0".."5"`（下载状态）、`if src == "..."` 链 |
| 分支 | ≥3 个同值比较用 `switch` | 长 `if - else if` 链 |
| 集合变换 | `.map` / `.filter` / `.reduce` | `Array[x] out = []` + `for` + `add` |
| 错误 | 项目内 `Result` struct | `catch e` 只 `print` |

已经存在、不要手写的（标准库 / 本项目薄封装）：

```leno
strings.pad_start(_str(v), 2, "0")   // 补零（本项目的 pad2 就是它）
strings.split(s, ",")                // 切分
strings.lines(s)                     // 按行切
strings.fmt_size(1536)               // "1 KB"
arr.sort(func(int a, int b) { return b - a })   // 自定义比较器（稳定归并）
maths.pow_mod(base, exp, m)          // 模幂（crypto_math 的 powmod 已改用它）
jsons.encode(obj)                    // 生成 JSON —— **不要手拼字符串**
```

**关键事实（实测）**：`Array[struct]` **支持原地更新**（`tasks[0].status = 1`、
`for tasks to t { t.status = 2 }` 都生效），不要为了"能改字段"而退回字符串行。

## 遗留（不要扩散）

- 行协议 + `F_*` 列号常量（`dl_engine.leno` 等）—— 新表一律 `Array[struct]`
- `insert_sorted()` 是**二分插入**（O(log n)），**不要**改成全量 `sort`（会退化成 O(n² log n)）
- 以前手写的 `jsonescape` / `padleft` / `powmod` 都已删，别再写回来

## 自检

```powershell
..\Leno\build\leno.exe --check xxx.leno           # 单文件
Get-ChildItem test -Filter *.leno | % { ..\Leno\build\leno.exe $_.FullName }   # 17 个离线测试
```

UI 部分需要窗口；纯逻辑（`player.leno` / `parsing.leno` / `crypto_math.leno`）刻意不 import SDL3，
**新增逻辑优先放这一层**，才能离线测。
