
# PMC 批量文献下载指南 (2026过渡版)

## 一、项目背景与重要通知
### 1.1 NCBI的公告
根据 NCBI 2026 年 4 月 13 日的公告：
*   **架构调整**：所有旧版 FTP 文件已移动到 `/deprecated/` 路径下。
*   **最后期限**：旧版路径将于 **2026 年 8 月** 彻底删除，之后必须转向 AWS S3 云服务。
*   **下载限制**：NCBI 对并发连接有严格限制，过高频率会导致 **503 Service Unavailable** 错误。
**公告链接**：https://pmc.ncbi.nlm.nih.gov/tools/ftp/#pad2026

### 1.2 PMC中能够免费下载的文献
**并不是 PMC 上的所有文章都能批量下载！**

能免费且批量下载的文章为 OA Subset (Open Access Subset)中的如下文章：
- **Commercial Use Subset**: 允许商业使用的 OA 文献。
- **Non-Commercial Use Subset**: 仅限非商业使用的 OA 文献。
- **作者手稿 (Author Manuscripts)**: 只有部分可以获取。

在批量下载 PDF 之前，你必须先下载一个“索引表”，这个表告诉你每一篇文献在服务器的哪个位置。
- **索引文件地址**： ftp://ftp.ncbi.nlm.nih.gov/pub/pmc/oa_file_list.csv
- **表格内容**： 包含 Accession ID (PMC号)、File Path (服务器路径)、PMID 等。

## 二、环境准备
建议在 Linux (Ubuntu/CentOS) 环境下执行，需安装以下工具：
*   `wget` 或 `aria2`
*   `grep`, `sed`, `awk`
*   `python3` (可选，用于大数据处理)

## 三、操作流程

### 第一步：导出目标 PMCID 列表
1.  访问 [PMC 搜索页面](https://pmc.ncbi.nlm.nih.gov/search/)，输入关键词（如 `deep laerning`）。
<img width="1379" height="691" alt="image" src="https://github.com/user-attachments/assets/93805dfb-c491-4f98-8049-ee9dcbd995ac" />

2.  设置过滤器（如 `2000-2026`）。
3.  点击 **Save** -> Selection: **All results** -> Format: **PMCID List**。
<img width="688" height="464" alt="image" src="https://github.com/user-attachments/assets/847b468d-00dd-48fa-98c9-17e1f0418525" />

4.  将下载的文件重命名为 `pmcid-list.txt`。

### 第二步：下载索引元数据 (Index File)
下载最新的 OA 文献索引表，该表包含所有文献的服务器路径。
```bash
# 由于文件较大 (约 900MB+)，建议使用断点续传，不要使用过大的并发数，会被ban IP
aria2c -x 1 -s 1 --min-split-size=100M -c https://ftp.ncbi.nlm.nih.gov/pub/pmc/deprecated/oa_file_list.csv

# wget太慢了！
# wget -c https://ftp.ncbi.nlm.nih.gov/pub/pmc/deprecated/oa_file_list.csv
```


### 第三步：预处理 ID 列表
清理 Windows 换行符，防止匹配失败。
```bash
sed -i 's/\r//g' pmcid-list.txt
```

### 第四步：路径匹配与链接生成
从 900MB 的大表中提取目标文献的下载路径，并拼接完整 URL。

**使用 grep 进行路径匹配与链接生成**
```bash
# 匹配 ID 并提取路径列，拼接 deprecated 路径
grep -F -f pmcid-list.txt oa_file_list.csv | awk -F ',' '{print "https://ftp.ncbi.nlm.nih.gov/pub/pmc/deprecated/" $1}' > final_links.txt
```

### 第五步：执行批量下载
使用 `wget` 进行下载，务必限制速率以防封锁 IP。
```bash
wget -i final_links.txt \
     -c \
     -nc \
     --limit-rate=1m \
     --wait=1 \
     --random-wait
```
*   `-c`: 断点续传。
*   `-nc`: 不覆盖已存在文件。
*   `--limit-rate=1m`: 限制每秒 1MB。

## 3. 后续处理

### 格式说明
*   如果路径包含 `oa_pdf/`：下载的是 **.pdf** 纯文件。
*   如果路径包含 `oa_package/`：下载的是 **.tar.gz** 压缩包，内含 XML、图片及 PDF。

### 提取 PDF (针对 .tar.gz 包)
下载完成后，可一键提取所有 PDF 到统一目录：
```bash
mkdir -p extracted_pdfs
for f in *.tar.gz; do
    tar -xzvf "$f" --wildcards "*.pdf" --transform='s|.*/||' -C ./extracted_pdfs
done
```

## 4. 故障排查
*   **503 Error**: 连接数过多。请将 `wget` 并发减至 1，或增加 `--wait` 时间。
*   **文件不存在**: 检查链接中是否包含 `deprecated`。2026 年 4 月后的旧版文件必须包含此路径。
*   **匹配不到**: 检查 `oa_file_list.csv` 的表头，确认 PMCID 所在的列索引是否正确。

---

**维护者建议**：请于 2026 年 8 月前完成所有旧数据的迁移。在此之后，请学习使用 `aws s3 sync` 命令直接从 `s3://pmc-oa-opendata/` 同步新版数据。
