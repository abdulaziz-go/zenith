# HLD: Zenith ETL (High-Performance Zero-Copy Edition)

## 1. Executive Summary
**Zenith** — bu MongoDB-dan ClickHouse-ga ma'lumotlarni o'ta yuqori tezlikda ko'chirish uchun mo'ljallangan ETL vositasi. Tizim xotira samaradorligini oshirish uchun **Apache Arrow** formatidan foydalanadi va Golang'da **Zero-copy** tamoyili asosida ishlaydi. Ma'lumotlarni qayta ishlash "Kernel Space" ga yaqin darajada (minimal context switch) amalga oshirilib, to'g'ridan-to'g'ri network interface orqali ClickHouse'ga uzatiladi.

## 2. System Architecture & Epics
* **Control Plane (SQLite):** Konfiguratsiya va sinxronizatsiya holatini (state) boshqarish.
* **Data Plane (Go + Apache Arrow):** Ma'lumotlarni oqim ko'rinishida olish, transformatsiya va yuklash.
* **Kernel-Space Optimization:** User space va kernel space o'rtasidagi ma'lumot ko'chirishni (buffer copying) minimallashtirish.
* **Sync Strategies:** Incremental Stream va Full Append (Truncate & Reload).

## 3. High-Performance Data Pipeline (The "Zero-Copy" Flow)

Zenith an'anaviy ETL vositalari kabi har bir JSON obyektini o'zgaruvchilarga deserialize qilmaydi. Buning o'rniga quyidagi mantiq qo'llaniladi:

### 3.1 Apache Arrow Integration
Ma'lumotlar MongoDB-dan olingandan so'ng, darhol **Apache Arrow (In-memory Columnar Format)** strukturasiga joylashtiriladi.
* **Zero-copy:** Arrow xotira layouti shunday tuzilganki, Golang'dagi ma'lumotlar bufferi va network socket'ga uzatiladigan byte oqimi bir xil ko'rinishda bo'ladi.
* **Vectorized Processing:** Transformatsiya mantiqi satrma-satr emas, balki ustunlar bo'yicha (SIMD optimization) bajariladi.

### 3.2 Memory & Network Path
1. **Extract:** MongoDB driver orqali olingan byte-stream xotiraga (Off-heap memory) joylanadi.
2. **Transform:** Apache Arrow C-data interface yoki Go native implementation yordamida ma'lumotlar tabular (ustunli) formatga o'tkaziladi.
3. **Load (Network Direct):** `sendfile(2)` yoki Golang'dagi `io.Copy` va `WriteTo` metodlari orqali, kernel bufferidagi ma'lumotlar foydalanuvchi ilovasi (User-space) ichida qayta nusxalanmasdan, to'g'ridan-to'g'ri ClickHouse TCP socket'iga uzatiladi.



## 4. Technical Requirements & Components

### 4.1 Sync Engine (Golang)
| Component | Function | Implementation Detail |
| :--- | :--- | :--- |
| **BSON-to-Arrow Converter** | MongoDB hujjatlarini Arrow Record Batch'ga o'tkazish. | CGO yoki Go-native Arrow builder |
| **Stream Manager** | `Incremental` yoki `Full Append` rejimlari boshqaruvi. | `TRUNCATE` logic for Full Append |
| **Zero-Copy Transporter** | Arrow bufferlarini to'g'ridan-to'g'ri socketga yozish. | `syscall.Sendfile` yoki `net.Buffers` |

### 4.2 State Store (SQLite)
Zenith o'zining "miyasi" sifatida SQLite'dan foydalanadi:
* **Storage:** 2-5 MB RAM footprint.
* **Resilience:** Har bir muvaffaqiyatli Arrow Batch yuborilgandan so'ng, `checkpoint` SQLite'da saqlanadi.

## 5. Synchronization Strategies

### 5.1 Incremental Sync (Streaming)
* **Logic:** MongoDB `Change Streams` yoki `_id` cursor orqali faqat delta'ni o'qiydi.
* **Checkpoints:** SQLite'dagi `last_offset` yordamida ishni to'xtagan joyidan davom ettiradi.

### 5.2 Full Append (Atomic Reload)
* **Logic:** ClickHouse'dagi eski ma'lumotlar o'chiriladi va to'liq qayta yuklanadi.
* **Performance:** Arrow formatida barcha ma'lumotlar "chunk" (paket) ko'rinishida parallel oqimlarda (Go routines) yuboriladi.

## 6. System Components Table

| Component | Function | Comment |
| :--- | :--- | :--- |
| **Zenith Core Engine** | Go routines orqali parallel ETL jarayonlarini boshqaradi. | Ultra-lightweight binary |
| **Arrow Buffer Pool** | RAMni tejash uchun Arrow bufferlarini qayta ishlatadi (GC bosimini kamaytiradi). | Memory management |
| **SQLite DB** | Metadata, Connectors va Job History. | Persistent & Local |
| **ClickHouse Native TCP** | Arrow byte-streamini qabul qiluvchi interfeys. | Direct write |

## 7. Performance Targets
* **Throughput:** Sekundiga 100,000+ qator sinxronizatsiyasi.
* **Latency:** < 1 soniya (Incremental mode).
* **RAM Usage:** SQLite (~2-5MB) + Arrow Buffer (Configurable, e.g., 64MB).

## 8. Deployment Model
Zenith bitta konteyner (Docker) yoki binary sifatida ishlaydi. SQLite bazasi konteyner ichidagi volume'da saqlanadi, bu esa "Zero-config" ETL tajribasini beradi.