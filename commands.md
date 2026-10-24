# คู่มือคำสั่ง: E. coli Genome Assembly

คู่มือนี้รวบรวมคำสั่งสำหรับการประกอบจีโนมแบคทีเรียจากข้อมูล Oxford Nanopore long reads โดยใช้ข้อมูล SRA accession `SRR27502765`

> หมายเหตุ: คำสั่งในเอกสารนี้เป็นคู่มือสำหรับทำซ้ำ ไม่จำเป็นต้องรันทั้งหมดใหม่ หากมีผลวิเคราะห์เดิมอยู่แล้ว

## 1. ตรวจสอบตำแหน่งและไฟล์

ตรวจสอบโฟลเดอร์ปัจจุบัน:

```bash
pwd
```

ดูไฟล์และโฟลเดอร์:

```bash
ls -lah
```

## 2. เตรียม working directory

สร้างโฟลเดอร์สำหรับเก็บข้อมูลวิเคราะห์:

```bash
mkdir -p ~/ecoli-SRR27502765
cd ~/ecoli-SRR27502765
```

สร้างโฟลเดอร์สำหรับ reads ที่ผ่านการกรองและผลตรวจคุณภาพ:

```bash
mkdir -p filtered qc_raw
```

ตรวจสอบตำแหน่ง:

```bash
pwd
ls -lah
```

**ผลที่ควรได้:** อยู่ใน `~/ecoli-SRR27502765` และมีโฟลเดอร์ `filtered` กับ `qc_raw`

## 3. ดาวน์โหลดข้อมูล sequencing จาก NCBI SRA

ตรวจสอบ accession ก่อนดาวน์โหลด:

```bash
prefetch --dry-run SRR27502765
```

ดาวน์โหลดข้อมูล:

```bash
prefetch SRR27502765 --output-directory ~/ecoli-SRR27502765
```

เปลี่ยนไปยัง working directory:

```bash
cd ~/ecoli-SRR27502765
```

แปลงไฟล์ SRA เป็น FASTQ:

```bash
fasterq-dump -e 4 --outdir . SRR27502765/SRR27502765.sra
```

ตรวจสอบไฟล์ที่ได้:

```bash
ls -lh SRR27502765.fastq
```

แปลงไฟล์ FASTQ เป็นไฟล์บีบอัด เพื่อลดพื้นที่จัดเก็บ:

```bash
gzip SRR27502765.fastq
```

ตรวจสอบไฟล์หลังบีบอัด:

```bash
ls -lh SRR27502765.fastq.gz
```

**ผลที่ควรได้:** มีไฟล์ `SRR27502765.fastq.gz` ซึ่งจะถูกนำไปใช้ในขั้นตอนถัดไป
> หมายเหตุ: ขั้นตอนนี้ใช้พื้นที่ดิสก์ค่อนข้างมาก ควรตรวจสอบพื้นที่ว่างก่อนดาวน์โหลดหรือแปลงไฟล์

## 4. ตรวจคุณภาพ reads ก่อนกรอง (Raw-read quality control)

กลับไปยัง working directory และสร้างโฟลเดอร์ผลตรวจ:

```bash
cd ~/ecoli-SRR27502765
mkdir -p qc_raw
```

ใช้ FastQC ตรวจคุณภาพ reads:

```bash
fastqc --memory 4096 --threads 1 \
  --outdir qc_raw SRR27502765.fastq.gz
```

ตรวจไฟล์รายงาน:

```bash
ls -lh qc_raw/
```

FastQC สร้างรายงาน HTML สำหรับเปิดดูในเว็บเบราว์เซอร์ และไฟล์ ZIP ที่มีผลวิเคราะห์รายละเอียด

## 5. กรอง reads ด้วย Chopper (Read filtering)

Chopper ใช้กรอง reads ตามคุณภาพและความยาว ในการวิเคราะห์นี้กำหนด quality threshold ที่ 10 และความยาวขั้นต่ำ 1,000 bp

```bash
cd ~/ecoli-SRR27502765

mkdir -p filtered

gzip -dc SRR27502765.fastq.gz \
  | chopper -q 10 -l 1000 \
  > filtered/SRR27502765_filtered.fastq
```

ตรวจสอบไฟล์ผลลัพธ์:

```bash
ls -lh filtered/SRR27502765_filtered.fastq
```

> หมายเหตุ: `-q 10` คือค่าเฉลี่ยคุณภาพขั้นต่ำที่ใช้กรอง และ `-l 1000` คือความยาวขั้นต่ำ 1,000 bp ตามตัวเลือกของ Chopper

## 6. ประกอบจีโนมด้วย Flye (Genome assembly)

Flye ใช้ long reads ที่ผ่านการกรองเพื่อสร้าง contigs เบื้องต้น ในการวิเคราะห์นี้กำหนดขนาดจีโนมโดยประมาณที่ 5 Mb และใช้ 4 threads

```bash
cd ~/ecoli-SRR27502765

flye \
  --nano-raw filtered/SRR27502765_filtered.fastq \
  --out-dir flye_output \
  --genome-size 5m \
  --threads 4
```

ตรวจสอบไฟล์ผลลัพธ์:

```bash
ls -lh flye_output/
```

ดูข้อมูลสรุป contigs:

```bash
seqkit stats flye_output/assembly.fasta
```

อ่านข้อมูลประกอบการประกอบจีโนม:

```bash
cat flye_output/assembly_info.txt
```

ผลหลักของขั้นตอนนี้คือ `flye_output/assembly.fasta` ซึ่งเป็น draft assembly ที่จะใช้เป็นจุดเริ่มต้นสำหรับการ polishing ในขั้นตอนถัดไป

## 7. Polishing รอบแรกด้วย Racon

Racon ใช้ reads จัดแนวกับ draft assembly เพื่อปรับลำดับเบสใน assembly ตามหลักฐานจาก reads

### 7.1 จัดแนว reads กับ Flye assembly

```bash
cd ~/ecoli-SRR27502765

minimap2 -x map-ont -t 4 \
  flye_output/assembly.fasta \
  SRR27502765.fastq.gz \
  > racon_round1.paf
```

### 7.2 สร้าง polished assembly

```bash
racon -t 4 \
  SRR27502765.fastq.gz \
  racon_round1.paf \
  flye_output/assembly.fasta \
  > assembly_racon1.fasta
```

ตรวจสอบผลลัพธ์:

```bash
seqkit stats assembly_racon1.fasta
```

> หมายเหตุ: ขั้นตอนนี้ใช้ raw reads ตาม workflow ที่ทำจริง ไม่ใช่ reads ที่ผ่าน Chopper

## 8. Polishing ด้วย Medaka

Medaka ใช้ neural-network-based polishing เพื่อปรับ consensus sequence โดยอาศัยข้อมูล reads และ draft assembly

ใน workflow เดิมนี้ Medaka ใช้ raw reads เป็น input และใช้ Flye assembly เป็น draft ด้วยตัวเลือก `-d` ไม่ใช่ `assembly_racon1.fasta`

```bash
cd ~/ecoli-SRR27502765

medaka_consensus \
  -i SRR27502765.fastq.gz \
  -d flye_output/assembly.fasta \
  -o medaka_output \
  -m r941_min_sup_g507 \
  -t 4
```

ตรวจสอบไฟล์ consensus:

```bash
ls -lh medaka_output/
seqkit stats medaka_output/consensus.fasta
```

> ข้อควรระวัง: ต้องเลือก Medaka model ให้ตรงกับ basecaller, chemistry และข้อมูล sequencing ที่ใช้จริง ชื่อโมเดลข้างต้นบันทึกตาม workflow เดิม แต่ไม่ควรถือว่าเป็นการยืนยัน basecaller จาก FASTQ เพียงอย่างเดียว

## 9. เปรียบเทียบ assembly ด้วย QUAST

QUAST ใช้สรุปตัวชี้วัดของ assembly เช่น จำนวน contigs ความยาวรวม ความยาว contig ที่ใหญ่ที่สุด และ N50

### 9.1 เปรียบเทียบ Flye กับ Racon

```bash id="0q27fy"
cd ~/ecoli-SRR27502765

quast \
  flye_output/assembly.fasta \
  assembly_racon1.fasta \
  -o quast_comparison \
  -t 4 \
  --min-contig 0
```

### 9.2 เปรียบเทียบ Flye, Racon และ Medaka

```bash id="80j5h7"
quast \
  flye_output/assembly.fasta \
  assembly_racon1.fasta \
  medaka_output/consensus.fasta \
  -o quast_medaka_comparison \
  -t 4 \
  --min-contig 0
```

ดูรายงาน:

```bash id="7d18y9"
cat quast_medaka_comparison/report.txt
```

> การเปรียบเทียบ QUAST ช่วยดูคุณลักษณะของ assembly แต่หากไม่มี reference genome ที่เชื่อถือได้ จะยังสรุปไม่ได้จากความยาว contig เพียงอย่างเดียวว่า assembly ใดมีความถูกต้องทางลำดับเบสมากที่สุด

## 10. ตรวจสอบการจัดแนว reads และ coverage

ตัวอย่างนี้จัดแนว raw reads กับ Medaka consensus แล้วสร้างไฟล์ BAM ที่เรียงลำดับแล้ว

```bash id="4n0o8m"
cd ~/ecoli-SRR27502765

minimap2 -ax map-ont -t 4 \
  medaka_output/consensus.fasta \
  SRR27502765.fastq.gz \
  | samtools sort -@ 2 -o medaka_mapped.bam
```

สร้างดัชนี BAM:

```bash id="8t7j7a"
samtools index medaka_mapped.bam
```

สรุปสถิติการจัดแนว:

```bash id="p8x9bs"
samtools stats medaka_mapped.bam > medaka_mapping_stats.txt

grep -E '^SN     (reads mapped|bases mapped \(cigar\)|mismatches|error rate):' \
  medaka_mapping_stats.txt
```

ตรวจ coverage ของแต่ละ contig:

```bash id="2a4h5n"
samtools coverage medaka_mapped.bam
```

> ข้อควรระวัง: reads ชุดเดียวกับที่ใช้สร้างหรือ polish assembly ไม่ใช่หลักฐานอิสระสำหรับยืนยันความถูกต้องทั้งหมด และ mapping error rate เพียงอย่างเดียวไม่ใช่ข้อพิสูจน์ว่า polishing ทำให้ assembly ดีขึ้น

## 11. ตรวจสอบความยาวและ GC content ด้วย SeqKit

สรุปจำนวนลำดับและความยาว:

```bash id="j2y0nc"
seqkit stats \
  flye_output/assembly.fasta \
  assembly_racon1.fasta \
  medaka_output/consensus.fasta
```

ดูความยาวและ GC content ของแต่ละ contig:

```bash id="z5n1p7"
seqkit fx2tab -n -l -g medaka_output/consensus.fasta
```

เก็บรายงาน QUAST ที่เปรียบเทียบครบทั้งสาม assemblies แยกจากรายงานเดิม:

```bash id="t3s8v4"
cp quast_medaka_comparison/report.txt \
  ~/ecoli-SRR27502765-github/results/quast_medaka_comparison.txt

cp quast_medaka_comparison/report.tsv \
  ~/ecoli-SRR27502765-github/results/quast_medaka_comparison.tsv
```

