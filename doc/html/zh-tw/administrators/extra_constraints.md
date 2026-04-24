# Slurm 額外條件約束

## TL;DR

可以為節點設定 JSON 格式的額外資料（Extra Data），讓作業透過 `--extra` 選項篩選節點，需在 `slurm.conf` 啟用 `SchedulerParameters=extra_constraints`。查詢語法支援比較運算子與布林運算子組合，但同一層括號內的布林運算子必須一致。注意：此功能會降低回填排程器（backfill scheduler）的預測準確性。

---

## 翻譯

### 概觀

可以為節點新增額外資料，作業也可以提出額外條件約束（extra constraints），依據節點的額外資料來篩選節點。此功能預設停用，可在 `slurm.conf` 中啟用。**警告**：Slurm 的回填排程器（backfill scheduler）無法針對額外條件約束尚未立即滿足的作業正確規劃節點。這表示節點的額外資料變動越頻繁，回填排程器的準確性就越低。

---

### 設定

在 `slurm.conf` 中設定：

```
SchedulerParameters=extra_constraints
```

---

### 節點額外資料

節點的額外資料（Extra Data）是一個 JSON 格式的字串。可在 `slurmd` 啟動時透過 `--extra` 旗標初始化。例如：

```
slurmd --extra '{ "a": 1.23, "b": true, "c": 0, "foo": "bar", "zed": 23 }'
```

也可以用 `scontrol` 更新。例如：

```
scontrol update nodename=node123 extra='{ "a": 1.23, "b": true, "c": 0, "foo": "bar", "zed": 23 }'
```

這會定義可供 `salloc`、`sbatch`、`srun` 的 `--extra` 選項請求的特性。值可以是任意字串、數字或布林值。

---

### 作業提交

#### 語法

`salloc`、`sbatch`、`srun` 的 `--extra` 欄位是任意字串；若包含空白或某些特殊字元，需用單引號或雙引號括住。

若已啟用 `SchedulerParameters=extra_constraints`，此字串會根據各節點的 *Extra* 欄位進行節點篩選。

最基本的請求結構如下：

```
<key><comparison_operator><value>
```

鍵（key）和值（value）是任意的非空字串，不能包含運算子的組成字元，也不能包含括號。因此，以下字元不允許出現在鍵或值中：

```
,&|<>=!()
```

允許使用的比較運算子如下：

- `=`（等於）
- `!=`（不等於）
- `>`（大於）
- `>=`（大於或等於）
- `<`（小於）
- `<=`（小於或等於）

若兩個數字的差小於 0.00001，則視為相等。不支援數字後綴（如 `kb`、`mb`）。若字母與數字混合，則鍵或值視為字串。

多個請求可用布林運算子串接：

```
<request><boolean_operator><request>
```

允許使用的布林運算子如下：

```
&   (AND)
,   (AND)
|   (OR)
```

可用任意數量的括號將請求分組。**同一層括號內的所有布林運算子必須相同。** 不同層括號可使用不同的布林運算子。

例如，以下是**不允許**的寫法：

```
a=1&b=2|c=foobar
```

但以下是允許的：

```
(a=1&b=2)|c=foobar
```

#### 注意事項

空白字元不作特殊處理，會被視為鍵或值的一部分。這表示以下寫法是**無效的**：

```
--extra " (a=b)"
```

開頭的空白會被解析為請求的鍵，接著的左括號字元既不是有效的鍵，也不是有效的比較運算子。此請求會導致作業被拒絕。

但以下寫法是有效的：

```
--extra "( a=b)"
```

這包含一個請求，鍵為 `" a"`（含一個前置空白），比較運算子為 `=`，值為 `b`。

同樣的注意事項也適用於單引號和雙引號——這些都不視為特殊字元，因此是字串的一部分。也就是說，`bar` 和 `"bar"` 並不相等。

#### 有效與無效的請求

以下是**有效**請求的範例：

```
a=1.23
a=   b
a!=1.24
a!=1.23|foo!=blah
b=200
b=true
foo<baz
(c<=0.0001&a=1.25)|zed=23.0
((c<=0.0001&a=1.25)|zed=23.0)&(a<1|b=false|c>=0.00000001)
((c<=0.0001&a=1.25)|zed=23.0)&(a<1|b=true|c>=0.1)
```

以下是**無效**請求的範例：

比較運算子無效：

```
a,<=6
```

結尾多餘運算子：

```
a<=6<=
```

連續多個布林運算子：

```
a=5&&&b=5
a=5|||b=5
```

連續多個比較運算子：

```
a====5
b<=<=5
```

括號內沒有任何內容：

```
a=5&()
```

同一層括號內使用不同的布林運算子：

```
a=5&b=5|c=5
(a=1)&(b=2)|(c=3)
```

請求之間沒有布林運算子：

```
a=1(b=2)
(a=1)(b=2)
(((a=1)b=2))
```

---

### 範例

給定一個具有以下額外資料的節點：

```
Extra={ "a": 1.23, "b": true, "c": 0, "foo": "bar", "zed": 23 }
```

以下 `--extra` 請求可由此節點滿足：

```
a=1.23
a!=1.24
a!=1.23|foo!=blah
b=200
b=true
foo<baz
(c<=0.0001&a=1.25)|zed=23.0
((c<=0.0001&a=1.25)|zed=23.0)&(a<1|b=false|c>=0.00000001)
((c<=0.0001&a=1.25)|zed=23.0)&(a<1|b=true|c>=0.1)
```

以下 `--extra` 請求**無法**由此節點滿足：

```
a!=1.23
b=0
b=false
foo>baz
((c<=0.0001&a=1.25)|zed=23.0)&(a<1|b=false|c>=0.00001)
```

提醒：兩個數字被視為相等的條件是差值必須小於 0.0001。這就是為什麼 `0.0001` 不被視為等於 `0`，因此請求 `c>=0.0001` 不被滿足；但 `0.00000001` 被視為等於 `0`，因此請求 `c>=0.00000001` 可以被滿足。

---

實際應用範例：可以撰寫一個腳本，監看每個節點的平均負載（load average），並持續用當前數值更新各節點的 `extra` 屬性，讓使用者將作業限制在負載低於某個門檻值的節點上。

在以下簡單範例中，叢集內三個節點正在被監控，`extra` 屬性被填入其平均負載：

```
$ scontrol show nodes node[01-03] | grep -E 'NodeName|Extra'
NodeName=node01 Arch=x86_64 CoresPerSocket=6
   Extra={ "load": 0.99 }
NodeName=node02 Arch=x86_64 CoresPerSocket=6
   Extra={ "load": 0.75 }
NodeName=node03 Arch=x86_64 CoresPerSocket=6
   Extra={ "load": 0.45 }
```

作業可以要求執行在 CPU 使用率不到一半的機器上：

```
$ sbatch -n12 --extra "load<0.5" --wrap='srun sleep 10'
Submitted batch job 11206

$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
             11206     debug     wrap      ben  R       0:03      1 node03
```

作業也可以要求執行在負載落在可接受範圍內的節點上：

```
$ sbatch -n12 --extra "(load<0.9&load>0.5)" --wrap='srun sleep 10'
Submitted batch job 11207

$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
             11207     debug     wrap      ben  R       0:01      1 node02
```
