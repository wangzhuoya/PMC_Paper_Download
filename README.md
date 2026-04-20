# PMC 批量文献下载指南：PMC Bulk Article Download Guide 
(2026 Transition Version)

[中文文档](./readme_CN.md)

## 1. Project Background & Important Notices
### 1.1 NCBI Announcement
According to the NCBI announcement on April 13, 2026:
*   **Architectural Adjustment**: All legacy FTP files have been moved to the `/deprecated/` path.
*   **Deadline**: Legacy paths will be completely removed in **August 2026**, after which users must switch to AWS S3 cloud services.
*   **Download Limits**: NCBI has strict limits on concurrent connections; excessively high frequencies will result in a **503 Service Unavailable** error.
**Announcement Link**: https://pmc.ncbi.nlm.nih.gov/tools/ftp/#pad2026

### 1.2 Articles Available for Free Download on PMC
**Not all articles on PMC can be bulk downloaded!**

Articles available for free bulk download are restricted to the following within the **OA Subset (Open Access Subset)**:
- **Commercial Use Subset**: OA articles permitted for commercial use.
- **Non-Commercial Use Subset**: OA articles permitted for non-commercial use only.
- **Author Manuscripts**: Only some are accessible.

Before bulk downloading PDFs, you must first download an "index file" that tells you the location of each article on the server.
- **Index File Address**: ftp://ftp.ncbi.nlm.nih.gov/pub/pmc/oa_file_list.csv
- **Table Content**: Contains Accession ID (PMC ID), File Path (server path), PMID, etc.

## 2. Environment Setup
It is recommended to execute this in a Linux (Ubuntu/CentOS) environment. The following tools need to be installed:
*   `wget` or `aria2`
*   `grep`, `sed`, `awk`
*   `python3` (optional, for big data processing)

## 3. Workflow

### Step 1: Export Target PMCID List
1.  Visit the [PMC Search Page](https://pmc.ncbi.nlm.nih.gov/search/), and enter your keywords (e.g., `deep learning`).
<img width="1379" height="691" alt="image" src="https://github.com/user-attachments/assets/93805dfb-c491-4f98-8049-ee9dcbd995ac" />

2.  Set filters (e.g., `2000-2026`).
3.  Click **Save** -> Selection: **All results** -> Format: **PMCID List**.
<img width="688" height="464" alt="image" src="https://github.com/user-attachments/assets/847b468d-00dd-48fa-98c9-17e1f0418525" />

4.  Rename the downloaded file to `pmcid-list.txt`.

### Step 2: Download Index Metadata (Index File)
Download the latest OA article index file, which contains the server paths for all articles.
```bash
# Due to the large file size (approx. 900MB+), it is recommended to use resume-on-disconnect. Do not use high concurrency, or your IP will be banned.
aria2c -x 1 -s 1 --min-split-size=100M -c https://ftp.ncbi.nlm.nih.gov/pub/pmc/deprecated/oa_file_list.csv

# wget is too slow!
# wget -c https://ftp.ncbi.nlm.nih.gov/pub/pmc/deprecated/oa_file_list.csv
```

### Step 3: Preprocess ID List
Clean up Windows newline characters to prevent matching failures.
```bash
sed -i 's/\r//g' pmcid-list.txt
```

### Step 4: Path Matching and Link Generation
Extract the target articles' download paths from the large 900MB table and concatenate the complete URLs.

**Using grep for path matching and link generation**
```bash
# Match IDs, extract the path column, and concatenate the deprecated path
grep -F -f pmcid-list.txt oa_file_list.csv | awk -F ',' '{print "https://ftp.ncbi.nlm.nih.gov/pub/pmc/deprecated/" $1}' > final_links.txt
```

### Step 5: Execute Bulk Download
Use `wget` to perform the download. Be sure to limit the rate to prevent IP blocking.
```bash
wget -i final_links.txt \
     -c \
     -nc \
     --limit-rate=1m \
     --wait=1 \
     --random-wait
```
*   `-c`: Continue/resume partially downloaded file.
*   `-nc`: Do not overwrite existing files.
*   `--limit-rate=1m`: Limit download rate to 1MB per second.

## 4. Post-Processing

### Format Explanations
*   If the path contains `oa_pdf/`: The downloaded file is a pure **.pdf** file.
*   If the path contains `oa_package/`: The downloaded file is a **.tar.gz** archive containing XML, images, and the PDF.

### Extracting PDFs (for .tar.gz packages)
After downloading, you can extract all PDFs to a unified directory with a single command:
```bash
mkdir -p extracted_pdfs
for f in *.tar.gz; do
    tar -xzvf "$f" --wildcards "*.pdf" --transform='s|.*/||' -C ./extracted_pdfs
done
```

## 5. Troubleshooting
*   **503 Error**: Too many connections. Please reduce `wget` concurrency to 1, or increase the `--wait` time.
*   **File not found**: Check if the link contains `deprecated`. Legacy files after April 2026 must include this path.
*   **No matches found**: Check the header of `oa_file_list.csv` to ensure the column index for the PMCID is correct.

---

**Maintainer's Suggestion**: Please complete all legacy data migrations before August 2026. After that, please learn to use the `aws s3 sync` command to sync new data directly from `s3://pmc-oa-opendata/`.
