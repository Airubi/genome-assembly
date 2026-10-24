# การประกอบจีโนมแบคทีเรีย E. coli จากข้อมูล Oxford Nanopore Sequencing

## ภาพรวม

การประกอบจีโนม (*genome assembly*) ของแบคทีเรีย *Escherichia coli* สายพันธุ์ LCL001 จากข้อมูลการอ่านลำดับดีเอ็นเอแบบสายยาว (long reads) ด้วย Oxford Nanopore Sequencing ซึ่งดาวน์โหลดจาก NCBI Sequence Read Archive (SRA) หมายเลข SRR27502765

เนื้อหาครอบคลุมการประกอบจีโนมและการปรับปรุงลำดับเบส โดยเปรียบเทียบผลจากเครื่องมือ 3 ชนิด ได้แก่

1. **Flye** — ใช้ประกอบจีโนมเบื้องต้นแบบ de novo
2. **Racon** — ใช้ปรับปรุงลำดับเบสของจีโนมที่ประกอบได้ในขั้นแรก (consensus polishing)
3. **Medaka** — ใช้ปรับปรุง consensus sequence ด้วยแบบจำลองที่เลือกตามข้อมูลเกี่ยวกับเทคโนโลยีการหาลำดับเบสและวิธี basecalling ที่มีรายงานไว้

ใช้เครื่องมือผ่าน command line และเปรียบเทียบผลการประกอบจีโนมอย่างเป็นระบบ

## ข้อมูลที่ใช้

- สิ่งมีชีวิต: *Escherichia coli*
- สายพันธุ์: LCL001
- หมายเลข SRA: [SRR27502765](https://www.ncbi.nlm.nih.gov/sra/SRR27502765)
- แพลตฟอร์ม: Oxford Nanopore MinION
- Flow cell ที่มีรายงาน: R9.4.1
- ประเภทข้อมูล: Long-read sequencing

ข้อมูล FASTQ ที่ดาวน์โหลดมาประกอบด้วย 8,441 reads มีความยาวรวมประมาณ 215.4 ล้านเบส (215.4 Mb)

## ขั้นตอนการวิเคราะห์ (Workflow)

1. ตรวจสอบคุณภาพข้อมูลลำดับเบสด้วย FastQC
2. กรองข้อมูล reads ด้วย Chopper
3. ประกอบจีโนมเบื้องต้นแบบ de novo ด้วย Flye
4. จัดแนว reads เข้ากับ assembly ด้วย minimap2 และประมวลผลไฟล์ BAM ด้วย samtools
5. ปรับปรุง consensus sequence รอบแรกด้วย Racon
6. ปรับปรุง consensus sequence ด้วย Medaka
7. เปรียบเทียบคุณลักษณะของ assembly ด้วย QUAST

## สรุปผลการประกอบจีโนม

| ตัวชี้วัด | Flye | Racon รอบที่ 1 | Medaka |
|---|---:|---:|---:|
| จำนวน contigs | 2 | 2 | 2 |
| ความยาวรวม (bp) | 5,304,391 | 5,304,356 | 5,304,696 |
| Contig ที่ยาวที่สุด (bp) | 5,142,738 | 5,142,735 | 5,143,021 |
| GC content (%) | 50.71 | 50.71 | 50.71 |
| N50 (bp) | 5,142,738 | 5,142,735 | 5,143,021 |
| N90 (bp) | 5,142,738 | 5,142,735 | 5,143,021 |
| จำนวน N ต่อ 100 kbp | 0 | 0 | 0 |

ดูรายงานเปรียบเทียบจาก QUAST ได้ที่ไฟล์ `results/quast_comparison.tsv` และ `results/quast_comparison.txt`

## ผลการวิเคราะห์

ผลการประกอบจีโนมทั้งสามเวอร์ชันประกอบด้วย 2 contigs และมีความยาวรวมกับค่า GC content ใกล้เคียงกัน โดย assembly หลังผ่าน Medaka มีความยาวรวมมากกว่า assembly จาก Flye และ Racon เล็กน้อย

อย่างไรก็ตาม ตัวชี้วัดเหล่านี้อธิบายลักษณะโครงสร้างของ assembly เป็นหลัก ไม่สามารถใช้ยืนยันความถูกต้องของลำดับนิวคลีโอไทด์หรือความสมบูรณ์ทางชีววิทยาของจีโนมได้โดยลำพัง

## ข้อจำกัด

- Reads ที่ใช้จัดแนวเพื่อประเมิน assembly เป็นข้อมูลชุดเดียวกับที่ใช้ในการประกอบจีโนม ดังนั้นอัตราความผิดพลาดจากการ mapping จึงไม่ใช่การตรวจสอบความถูกต้องแบบอิสระ
- การเปรียบเทียบอัตราความผิดพลาดจาก mapping ไม่ได้แสดงให้เห็นว่าอัตราความผิดพลาดลดลงหลังการ polishing อย่างชัดเจน
- Contig ที่สองมี read depth ต่ำกว่า contig แรกอย่างมาก ยังไม่มีหลักฐานเพียงพอที่จะระบุเอกลักษณ์ทางชีววิทยา จึงยังไม่สรุปว่าเป็น plasmid
- ข้อมูล FASTQ headers ไม่ได้ยืนยันโดยตรงว่าใช้ basecaller รุ่นใดในการประมวลผลข้อมูลเดิม โดยเลือก Medaka model `r941_min_sup_g507` ตามข้อมูลที่รายงานเกี่ยวกับ flow cell R9.4.1 และ Guppy 5.0.7 SUP อย่างไรก็ตาม model นี้อาจไม่ตรงกับข้อมูลจริง หาก reads ที่ดาวน์โหลดมาผ่านการประมวลผลใหม่ด้วยวิธีอื่น
  
## โครงสร้างไฟล์

```text
results/
├── flye_assembly.fasta
├── racon_round1.fasta
├── medaka_consensus.fasta
├── quast_comparison.tsv
└── quast_comparison.txt
```

Repository นี้เก็บไฟล์ assembly ที่คัดเลือกแล้วและรายงานสรุปที่จำเป็น โดยไม่รวม raw reads และไฟล์ intermediate 

## เครื่องมือที่ใช้

- FastQC — ตรวจสอบคุณภาพ reads
- Chopper — กรอง reads
- Flye — ประกอบจีโนมแบบ de novo
- minimap2 — จัดแนวลำดับเบส
- samtools — จัดการและตรวจสอบข้อมูล alignment
- Racon — ปรับปรุง consensus sequence
- Medaka — ปรับปรุง consensus sequence ด้วย neural network models
- QUAST — ประเมินและเปรียบเทียบคุณลักษณะของ genome assembly
