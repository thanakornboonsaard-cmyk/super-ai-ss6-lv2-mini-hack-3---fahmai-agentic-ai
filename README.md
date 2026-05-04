# FahMai Directory Q&A — Answer Generation with Typhoon

โปรเจกต์นี้เป็นโน้ตบุ๊กสำหรับสร้างคำตอบอัตโนมัติให้โจทย์ **FahMai Directory Q&A** โดยใช้แนวทางผสมระหว่าง

- **rule-based / deterministic tools** สำหรับคำถามที่ตอบได้ตรงไปตรงมา
- **Typhoon v2.5 agentic harness** สำหรับบางกรณีที่ต้องใช้การจัดรูปคำตอบหรือ reasoning เพิ่มเติม
- **local grader + error analysis** เพื่อเช็กผลลัพธ์และวนปรับปรุงคำตอบได้ใน Colab

โน้ตบุ๊กหลัก:
- `Scamper03_fahmai_answer_generation_with_typhoon_final.ipynb`

---

## จุดประสงค์ของโน้ตบุ๊ก

โน้ตบุ๊กนี้ถูกออกแบบมาเพื่อ:

1. โหลดชุดข้อมูลพนักงาน คำถาม และไฟล์ตัวอย่าง submission
2. สร้าง index สำหรับค้นหาข้อมูลบุคคล หน่วยงาน สาขา อีเมล และเบอร์ติดต่อ
3. ตรวจจับคำถามที่ควร **refuse** ตามกติกา
4. route คำถามไปยัง tool ที่เหมาะสม
5. สร้างไฟล์ `submission.csv`
6. รัน `grade.py` ในเครื่องเพื่อประเมินคำตอบ
7. วิเคราะห์ข้อผิดพลาดบน public labels
8. ตรวจสอบว่าไฟล์ส่งออกพร้อมอัปโหลด และยืนยันการใช้ Typhoon

---

## โครงสร้างไฟล์ที่ต้องมี

ใน Google Drive ควรมีโครงสร้างประมาณนี้:

```text
My Drive/
└── super-ai-engineer-season-6-fahmai-2/
    ├── employees.csv
    ├── questions.csv
    ├── sample_submission.csv
    ├── train_labels.json
    ├── grade.py
    └── Submit/
```

> โน้ตบุ๊กมี logic ช่วย mount Google Drive และ fallback path ให้ แต่ควรเตรียมไฟล์ตามโครงสร้างนี้เพื่อให้รันได้ลื่นที่สุด

---

## ไฟล์อินพุต

### 1) `employees.csv`
ตารางข้อมูลพนักงาน ใช้เป็นฐานข้อมูลหลักสำหรับตอบคำถาม เช่น

- Employee ID
- แผนก / ฝ่าย / unit
- ชื่อไทย / อังกฤษ
- ชื่อเล่น
- เบอร์ต่อ
- เบอร์มือถือ
- อีเมล
- สาขา / office location
- ตำแหน่งงาน

### 2) `questions.csv`
ชุดคำถามสำหรับ inference โดยมีข้อมูลช่วย route บางส่วน เช่น

- `id`
- `language`
- `question`
- `hyphen_codes`
- `compact_codes`
- route-related helper columns

### 3) `sample_submission.csv`
template สำหรับสร้างไฟล์ส่ง โดยต้องมีคอลัมน์:

- `id`
- `response`

### 4) `train_labels.json`
ใช้สำหรับ local grading และ public-label error analysis

### 5) `grade.py`
สคริปต์ grader สำหรับประเมินผลคำตอบแบบ local

---

## สิ่งที่โน้ตบุ๊กทำในแต่ละส่วน

### Section 0 — Setup
ติดตั้งและ import library ที่จำเป็น รวมทั้ง utility พื้นฐานสำหรับ Colab / path / display / regex / json

### Section 1 — Mount Google Drive and define paths
พยายาม mount Google Drive แบบ robust และกำหนด path ของไฟล์หลัก เช่น

- `EMP_PATH`
- `Q_PATH`
- `SAMPLE_PATH`
- `TRAIN_LABELS_PATH`
- `GRADE_PATH`
- `SUBMIT_DIR`

### Section 2 — Load cleaned data
โหลด

- `employees.csv`
- `questions.csv`
- `sample_submission.csv`
- `train_labels.json`

โดยใช้ `dtype=str` และ `keep_default_na=False` เพื่อคุมรูปแบบข้อมูลให้เสถียร

### Section 3 — Sanity checks
ตรวจ schema และจำนวนแถวของข้อมูล เช่น

- employees = 1995
- questions = 300
- sample submission = 300

รวมถึงตรวจว่า required columns ครบ และ `id` ต่าง ๆ ไม่ซ้ำ

### Section 4 — Utility functions
รวม helper functions สำหรับ

- normalize text
- parse list-like columns
- lower/upper text
- title formatting
- text matching

### Section 5 — Parse question helper columns and fix routes
แปลงคอลัมน์พวก `hyphen_codes` / `compact_codes` ให้ใช้งานได้จริง และกำหนด runtime refusal detection เช่น

- field not allowed
- opinion / speculation
- prompt injection
- external company

### Section 6 — Refusal phrases
กำหนดข้อความปฏิเสธทั้งภาษาไทยและภาษาอังกฤษ เช่น

- `ไม่สามารถให้ข้อมูลนี้ได้`
- `ไม่พบข้อมูล`
- `ขอปฏิเสธคำขอ`

### Section 7 — Build lookup indexes
สร้าง dictionary / inverted indexes สำหรับค้นหาเร็ว เช่น

- ตาม Employee ID
- อีเมล
- เบอร์ต่อ
- เบอร์มือถือ
- หน่วยงาน / แผนก / section
- ชื่อจริง / ชื่อเล่น ไทยและอังกฤษ
- สาขา

### Section 8 — Formatting helpers
รวม helper สำหรับ format คำตอบให้อยู่ในรูปที่ grader ยอมรับได้ง่ายขึ้น

### Section 9 — Code and context extraction
สกัดข้อมูลบริบท เช่น branch code / org code / location context เพื่อช่วยตีความคำถาม

### Section 10 — Person search helpers
รวม logic สำหรับกรอง candidate records ตามบริบทของคำถาม เช่น

- branch
- department
- section
- unit
- org code

### Section 11 — Tools
นิยาม tool หลักหลายแบบ เช่น

- refusal tool
- reverse lookup จากเบอร์ต่อ / มือถือ / อีเมล
- secretary lookup
- role lookup
- org listing
- contact / email lookup
- nickname / person lookup

### Section 12 — Optional Typhoon formatter / API setup / harness
เตรียมการใช้ **Typhoon v2.5** ผ่าน Colab Secrets หรือ environment variables

ค่าที่โน้ตบุ๊กใช้งาน:
- `TYPHOON_API_KEY`
- `TYPHOON_API_URL`
- `TYPHOON_MODEL = "typhoon-v2.5-30b-a3b-instruct"`

สามารถเปิดหรือปิด harness ได้ผ่าน:

```python
USE_TYPHOON_HARNESS = True
```

> ถ้าเปิด harness แต่ไม่มี `TYPHOON_API_KEY` โน้ตบุ๊กจะหยุดด้วย error ทันที

### Section 13 — Answer dispatcher
เลือกว่าจะใช้ deterministic answer หรือ Typhoon harness ตาม route ของคำถาม

### Section 14 — Generate submission
วนตอบทุกคำถามใน `questions.csv` แล้วสร้าง

- `submission`
- `debug_df`

### Section 15 — Save submission
บันทึกไฟล์ผลลัพธ์ลงโฟลเดอร์ `Submit`

### Section 16 — Run local grader
รัน `grade.py` เพื่อตรวจผลลัพธ์ในเครื่อง

### Section 17 — Public label error analysis
วิเคราะห์ public-label failures แบบละเอียด เพื่อดูว่าแต่ละข้อพลาดเพราะอะไร

### Section 18 — Next iteration guide
สรุปแนวทางปรับปรุงรอบถัดไปตาม bucket ของข้อผิดพลาด

### Section 18.1 — Verify files in Submit folder
ตรวจว่ามีไฟล์ output ครบและ schema ถูกต้อง

### Section 18.2 — Confirm Typhoon usage
ตรวจจาก debug file ว่า harness ถูกเปิดจริงหรือไม่

---

## การตั้งค่า Typhoon API

### วิธีแนะนำ: Colab Secrets
เพิ่ม secret ชื่อ:

- `TYPHOON_API_KEY`

จากนั้นโน้ตบุ๊กจะโหลดเข้ามาอัตโนมัติ

### หรือกำหนดด้วย environment variable
```python
import os
os.environ["TYPHOON_API_KEY"] = "YOUR_KEY"
```

API URL ที่โน้ตบุ๊กตั้งไว้คือ:

```text
https://api.opentyphoon.ai/v1/chat/completions
```

---

## วิธีรัน

1. เปิดโน้ตบุ๊กใน Google Colab
2. mount Google Drive
3. ตรวจว่าไฟล์อินพุตอยู่ครบ
4. ตั้ง `TYPHOON_API_KEY` ถ้าต้องการใช้ harness
5. รันทุก section ตามลำดับ
6. ดูผลลัพธ์จาก local grader
7. เปิด error analysis เพื่อตรวจข้อที่ยังพลาด
8. ส่งไฟล์ submission ที่ได้ไปยังระบบแข่งขัน

---

## ไฟล์ output

หลังรันเสร็จ จะมีไฟล์สำคัญในโฟลเดอร์ `Submit/` เช่น

- `submission.csv` — ไฟล์สำหรับอัปโหลด
- `submission_debug_details_v6.csv` — debug details รายข้อ
- `public_error_analysis.csv` — วิเคราะห์ข้อผิดพลาดของ public labels

> ชื่อไฟล์จริงอาจขึ้นกับตัวแปร path ในโน้ตบุ๊กเวอร์ชันนั้น

---

## จุดเด่นของ approach นี้

- ใช้ **rule-based routing** ทำให้คำถามที่เป็น pattern ชัด ๆ ตอบได้เสถียร
- รองรับ **Thai / English**
- มี refusal handling ตามกติกา
- มี local grader ใน pipeline เดียวกัน
- มี error analysis ช่วย iteration
- สามารถเปิด **Typhoon harness** เฉพาะจุดที่ต้องการได้

---

## ข้อควรระวัง

- ถ้า path ของ Google Drive ไม่ตรงกับที่โน้ตบุ๊กคาดไว้ อาจอ่านไฟล์ไม่เจอ
- ถ้าเปิด `USE_TYPHOON_HARNESS=True` แต่ไม่มี API key จะรันไม่ผ่าน
- คำตอบบางประเภทต้องระวังมากเรื่อง
  - leaked employee ID
  - leaked phone extension
  - forbidden fields
  - prompt injection
- ข้อที่ deterministic ผ่านอยู่แล้ว ไม่ควรให้ LLM ไปเปลี่ยนคำตอบโดยไม่จำเป็น

---

## คำแนะนำในการปรับปรุงต่อ

ถ้าต้องการเพิ่มคะแนนหรือความเสถียรต่อ:

1. ปรับ refusal detection patterns ให้แม่นขึ้น
2. เพิ่ม alias / nickname coverage สำหรับ person search
3. ปรับ formatting helpers ให้เข้ากับ grader มากขึ้น
4. เพิ่ม routing rules ก่อน fallback ไป Typhoon
5. ใช้ debug file วิเคราะห์ว่าข้อไหนเรียก harness แล้วช่วยจริง
6. ดู bucket ใน error analysis แล้วแก้ทีละกลุ่ม

---

## สรุป

โน้ตบุ๊กนี้เป็น pipeline แบบครบวงจรสำหรับโจทย์ **FahMai Directory Q&A** ที่เน้น

- retrieval จาก directory data
- deterministic answer generation
- optional Typhoon augmentation
- local grading
- error analysis
- readiness check ก่อนส่งจริง

เหมาะสำหรับใช้ทั้งเป็น baseline ที่เสถียร และเป็นฐานสำหรับ iterate ต่อในรอบถัดไป
