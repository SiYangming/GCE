# 使用示例

以下示例中的 `path/to/` 请替换为实际路径；`gce` 与 `kmer_freq_hash` 已按 [INSTALL.md](INSTALL.md) 加入 `PATH`。

GCE 的典型链路分两步：先用 `kmer_freq_hash` 统计 reads 的 k-mer 频数，再用 `gce` 从频率谱估计基因组特征。

## 1. 生成 reads 文件列表

```bash
mkdir -p path/to/GCE
cd path/to/GCE

# reads.list 每行一个 FASTA/FASTQ 绝对路径
ls path/to/reads/illumina.?.fastq > reads.list
```

## 2. 计算 k-mer 频率

```bash
# -k 21      k-mer 长度
# -l         reads 路径列表
# -t 8       线程数
# -i 80000000 每线程初始 hash 表大小（估计的不同 k-mer 数）
# -o 0       不输出每条 k-mer 序列（省时）
# -p out     输出前缀 -> 生成 out.freq.gz / out.freq.stat
kmer_freq_hash -k 21 -l reads.list -t 8 -i 80000000 -o 0 -p out &> kmer_freq.log
```

## 3. 评估基因组特征

```bash
# -f out.freq.stat  k-mer 深度频率文件
# -c 21             unique k-mer 期望深度
# -g 273206457      总 k-mer 数（取自 kmer_freq.log 的 Kmer_individual_num）
# -m 1              连续模型
# -D 8              连续模型峰间距
# -b 1              存在测序偏好
gce -f out.freq.stat -c 21 -g 273206457 -m 1 -D 8 -b 1 > out.table 2> out.log
```

杂合基因组需额外加 `-H 1`：

```bash
gce -f out.freq.stat -c 21 -g 273206457 -m 1 -D 8 -b 1 -H 1 > out.h1.table 2> out.h1.log
```

## 4. 绘制 k-mer 分布图（可选）

```bash
# 转为两列（dep / num）格式
perl -e 'print "dep\tnum\n";while (<>) {print}' out.freq.stat > dep_num.txt

# 用 R + ggplot2 作图
echo 'a <- read.table("dep_num.txt", header=TRUE);
library("ggplot2")
png(file="dep_num.png", bg="transparent")
qplot(dep, num, data=a, ylim=c(0,3e5), xlim=c(0,180), geom=c("line"))
dev.off()
png(file="dep_individual.png", bg="transparent")
qplot(dep, num*dep, data=a, ylim=c(0,1e7), xlim=c(0,180), geom=c("line"))
dev.off()' | R --vanilla --slave
```

生成 `dep_num.png`（k-mer 深度频率分布）与 `dep_individual.png`（深度×频数分布），可据此判断主峰位置与是否存在杂合半峰。

## 5. 使用仓库自带测试数据（gce-1.0.2）

```bash
cd /path/to/install/GCE/gce-1.0.2/test/Achatina_fulica
cat work.sh          # 查看上游给出的运行方式
```

该目录内含 `AF.kmer.freq.stat`、`gce.table`、`gce.log` 等结果文件，可用于对照。

## 6. 参数说明

### kmer_freq_hash

| 参数 | 说明 |
| --- | --- |
| `-k` | k-mer 长度（9~27，默认 17） |
| `-l` | reads 路径列表（每行一个） |
| `-t` | 线程数 |
| `-i` | 每线程初始 hash 表大小 |
| `-o` | 是否输出每条 k-mer 序列（1=输出，0=不输出） |
| `-p` | 输出前缀 |

### gce

| 参数 | 说明 |
| --- | --- |
| `-f` | k-mer 深度频率文件（`.freq.stat`） |
| `-g` | 总 k-mer 数（建议设置，否则由频率文件推算，易因数据缺失产生误差） |
| `-c` | unique k-mer 期望深度（无清晰主峰或杂合度较高时建议设置） |
| `-m` | 估计模型：0=离散（默认）/ 1=连续（真实数据推荐） |
| `-D` | 连续模型峰间距（建议 `-D 8`，对应考虑 48 个峰） |
| `-b` | 是否存在测序偏好（1=有 / 0=无） |
| `-M` | 最大深度值，超过该深度的信息被忽略 |
| `-H` | 杂合模式（杂合基因组加 `-H 1 -c <unique_depth>`） |
| `-h` | 显示帮助 |

> `gce` 的结果表输出到 stdout（`> gce.table`），运行日志输出到 stderr（`2> gce.log`）。

## 7. 结果解读

最关键的估计结果在日志文件末尾的 `Final estimation table`：

```text
Final estimation table:
raw_peak  effective_kmer_species  effective_kmer_individuals  coverage_depth  genome_size  a[1]     b[1]
75        742400596               168346645871                75.8021         2.22087e+09  0.663012 0.271515
```

| 列 | 含义 |
| --- | --- |
| `raw_peak` | k-mer species 曲线上的主峰，对应非重复、非杂合区 |
| `effective_kmer_species` | 真实 k-mer 种类数（已排除测序错误导致的低频 k-mer） |
| `effective_kmer_individuals` | 真实 k-mer 个体数（已排除低频 k-mer） |
| `coverage_depth` | 真实 k-mer 的估计测序深度 |
| `genome_size` | 估计基因组大小（= effective_kmer_individuals / coverage_depth） |
| `a[1]` | 基因组中唯一 k-mer 占全部 k-mer 种类的比例 |
| `b[1]` | 基因组中唯一 k-mer 占全部 k-mer 个体的比例 |

杂合度估计（需 `-H 1 -c <unique_depth>`）：k-mer 杂合率 `KHR = a[1]/2/(2-a[1]/2)`，碱基杂合率 `BHR = KHR/k`（k 为 k-mer 长度）。

## 8. 注意事项

- `gce-1.0.0` 使用 kmerfreq 输出时需仅保留数据行、去掉头部注释行；`gce-alternative` 可直接使用带注释的 kmerfreq 输出。
- `gce-1.0.2` 兼容 kmerfreq 4.0（最大深度 65535）。
- 若主峰识别不准，可用 `-c` 指定 unique 峰所在深度，程序会在该深度附近重新识别峰。

## 9. 参考

Binghang Liu, Yujian Shi, Jianying Yuan, et al., Wei Fan*. Estimation of genomic characteristics by analyzing k-mer frequency in de novo genome projects. arXiv:1308.2012 (2013).
