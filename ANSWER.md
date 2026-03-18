# 改进前

原代码使用 `std::vector<bool> cells(N * N)`（N=4096），密集分配 **16,777,216** 个格子，
`step()` 每次用 OpenMP 并行遍历全部 16M 格子。实际活细胞约 3000 个，稀疏率仅 **0.018%**，
绝大多数遍历都是无用功。实测 **23.60s**（Apple M 系芯片，Release 模式，OpenMP 5.1）

# 改进后

```
step 0
left=1048, right=3225, count=2815
step 100
left=1051, right=3225, count=3290
step 200
left=1048, right=3223, count=2725
step 300
left=1048, right=3273, count=2925
step 400
left=1051, right=3323, count=3695
step 500
left=1048, right=3373, count=2760
step 600
left=1048, right=3423, count=3055
step 700
left=1051, right=3473, count=3630
left=1048, right=3523, count=2910   ← 与标准答案完全吻合
main: 0.477948s
```

# 加速比

原代码实测 **23.60s**，改进后 **0.478s**，加速约 **49x**。

理论复杂度从 O(N²) = O(16M) 降到 O(K) = O(~3000)，约 5000 倍理论加速。
实际加速比受限于哈希表常数开销，但仍远快于密集遍历。

---

# 实现方法

## 如何封装稀疏 Grid

使用 `std::unordered_set<int>` 存储活细胞坐标，key = `x * N + y`（`int` 32位）。
只有活细胞被存储，死细胞完全不占内存。
原来需要 `N×N = 16M` 个 bool（约 2MB），改造后只需存约 3000 个 int（约 12KB）。

## step() 的优化思路

原代码的 `step()` 遍历所有 N×N 格子，绝大多数都是死细胞，完全无意义。

新算法分两步：

**第一步：正向累加邻居计数**

遍历所有活细胞，对其 8 个邻居在 `unordered_map<int,int>` 中计数 +1。
只有"可能受影响"的格子才会出现在 map 里，天然过滤掉所有孤立死格子。

**第二步：应用 Conway 规则**

遍历 `neigh_count`，根据当前是否存活和邻居数决定下一代：
- 活细胞 && count == 2 或 3 → 继续存活
- 死细胞 && count == 3 → 变为活细胞
- 其余 → 死亡（不插入 next）

活细胞若邻居数为 0（孤立），根本不出现在 `neigh_count` 中，自动淘汰，无需特判。

## 有没有用位运算量化减轻内存带宽？

key = `x * N + y`，由于 N = 4096 = 2¹²，可等价为 `(x << 12) | y`，
用位运算代替乘除，`key_x(k) = k >> 12`，`key_y(k) = k & 0xFFF`，
减少 key 解析时的计算开销。

## Grid 是否可以并行访问？

当前实现为单线程，因为稀疏版本单线程已经比原来 OpenMP 并行的密集版本快得多。
若进一步优化，可将 `neigh_count` 的累加拆分为多线程 thread-local map 后 merge，
避免 hash map 写竞争（类似 TBB 的 `combinable` 思路）。

## 有没有用访问者模式缓存坐标？

使用 `.reserve()` 预分配哈希桶，减少 rehash 次数：
- `neigh_count.reserve(cells.size() * 9)` — 每个活细胞最多影响 8 个不同邻居格
- `next.reserve(cells.size() * 2)` — 下一代规模与当前代相近

---

# 我的创新点

- **自定义哈希函数** `CellHash`：乘以黄金比例常数 `2654435761u`（Knuth 乘法哈希），
  相比默认 `std::hash<int>` 对连续整数的冲突更少，减少链式探测开销。

- **"正向累加"算法**：相比"对每个候选格子逐一查询其邻居是否存活"的写法，
  正向累加只需遍历活细胞一次，cache 局部性更好，常数更小。

- **单一 int key 编码**：用 `int` 而不是 `pair<int,int>` 或 `tuple` 作 key，
  减少哈希结构体大小，降低内存和比较开销。