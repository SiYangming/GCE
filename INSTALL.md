# 安装与配置

GCE（Genome Characteristics Estimation）是基于 k-mer 频率的基因组特征评估工具，通过贝叶斯模型从 k-mer 深度频率谱估计基因组大小、重复序列比例与杂合度，用于组装前的基因组 survey。

仓库内包含三份代码，安装方式如下（路径请按实际环境替换）：

| 目录 | 版本 | 说明 |
| --- | --- | --- |
| `gce-1.0.0/` | 1.0.0 | 主程序 `gce` 带源码（`gce.cpp` + `Makefile`）与预编译 `gce` 二进制；`kmerfreq` 子程序为随包预编译二进制 |
| `gce-1.0.2/` | 1.0.2 | 主程序带源码，附测试数据；兼容 kmerfreq 4.0，上游建议一般用户使用该版本 |
| `gce-alternative/` | — | 另一套实现（Perl 脚本 + 两个 C++ 程序），可用于对照与验证 |

> 上游 `ReadMe.txt`（仓库根目录）为随包说明，保留了使用入口与版本说明；本文件补充安装与配置步骤。

## 环境要求

| 依赖 | 说明 |
| --- | --- |
| Linux x86_64 | 随包的 `gce` / `kmer_freq_hash` 为 x86_64 二进制 |
| g++ / make | 编译 `gce`（及 `gce-alternative` 内的 C++ 程序） |
| Perl | 运行 `gce-alternative/` 下的 Perl 脚本 |
| R + ggplot2 | 可选，用于绘制 k-mer 分布图 |

## 1. 获取代码

```bash
mkdir -p /path/to/install
cd /path/to/install

# 方式一：克隆仓库
git clone https://github.com/SiYangming/GCE.git /path/to/install/GCE

# 方式二：使用仓库 Release 附件 gce-1.0.0.tar.gz
# tar zxf gce-1.0.0.tar.gz -C /path/to/install/
```

以下步骤以 `gce-1.0.0` 为例（`gce-1.0.2` 步骤相同，替换目录名即可）。

## 2. 编译主程序 gce

```bash
cd /path/to/install/GCE/gce-1.0.0

# 仓库已带预编译二进制，可直接使用；如需重编译：
make        # g++ 编译 gce.cpp，生成 gce
```

## 3. kmerfreq 子程序

k-mer 计数由 `kmerfreq/kmer_freq_hash/` 下的 `kmer_freq_hash` 完成，仓库内已提供 x86_64 预编译二进制（无源码，无需编译）：

```bash
ls /path/to/install/GCE/gce-1.0.0/kmerfreq/kmer_freq_hash/
# ReadMe.txt  kmer_freq_hash
```

> 该目录内的 `ReadMe.txt` 记录了 `kmer_freq_hash` 的参数说明。

## 4. 配置环境变量

```bash
echo 'export PATH=/path/to/install/GCE/gce-1.0.0:$PATH' >> ~/.bashrc
echo 'export PATH=/path/to/install/GCE/gce-1.0.0/kmerfreq/kmer_freq_hash:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## 5. 验证安装

```bash
gce -h            # 打印 Version: 1.0.0
kmer_freq_hash    # 无参数运行可查看用法
```

## 6. 安装 gce-1.0.2（可选）

上游建议一般用户使用 1.0.2：

```bash
cd /path/to/install/GCE/gce-1.0.2
make              # 编译 gce
./gce -h
```

`gce-1.0.2/test/` 下附有两份测试数据（`Achatina_fulica`、`Achatina_immaculata`）及其运行脚本 `work.sh`，可用于核对结果。

## 7. 安装 gce-alternative（可选）

```bash
cd /path/to/install/GCE/gce-alternative

# 编译其中两个 C++ 程序
make              # 生成 estimate_multiple_poissons / estimate_repeat

# 其余为 Perl 脚本，直接以 perl 调用
```

## 8.（可选）安装 R 与 ggplot2

用于绘制 k-mer 分布图：

```bash
R -e 'install.packages("ggplot2")'
```

## 卸载

删除对应目录即可：

```bash
rm -rf /path/to/install/GCE
```

*注：GCE 为历史遗留软件（上游官方发布源已不可用），新项目建议改用 GenomeScope 2.0 + Jellyfish 做基因组 survey。*
