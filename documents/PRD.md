# PRD: Zenith (Ultra-Lightweight ETL Tool)

## 1. Goal
**Zenith** — bu resurslarni minimal darajada iste'mol qiladigan, yuqori samaradorlikka ega ETL vositasi. Loyihaning asosiy maqsadi MongoDB-dan ma'lumotlarni yig'ish, ularni transformatsiya qilish va ClickHouse omboriga sinxronizatsiya qilishdir. Tizimning barqarorligi va yengilligini ta'minlash uchun barcha konfiguratsiyalar, konnektorlar holati va metadata **SQLite** bazasida saqlanadi (SQLite resurs iste'moli ~2-5 MB atrofida saqlanishi maqsad qilingan).

---

## 2. Functional Requirements

### 2.1 Connector Management
* **Source Connector:** MongoDB ulanish sozlamalarini (Connection string, DB name, Collection) qo'shish va tahrirlash.
* **Destination Connector:** ClickHouse ulanish sozlamalarini (Host, Port, Credentials) boshqarish.
* **SQLite Persistence:** Barcha konnektorlar sozlamalari va metadata SQLite bazasida saqlanishi shart.

### 2.2 Data Transformation & Synchronization
* **Schema Mapping:** MongoDB-ning JSON (BSON) hujjatlarini ClickHouse-ning ustunli (tabular) tuzilmasiga o'tkazish.
* **Sync Interval:** Sinxronizatsiya davriyligini (masalan, har 60 sekundda) belgilash.
* **Incremental Sync:** Ma'lumotlarni faqat yangi qo'shilgan qismini (`_id` yoki `updated_at` bo'yicha) ko'chirish.
* **Full sync append:** Eski malumotlar update bo'lishi kutilyotgan tablelarni incremental sync o'rniga to'liq rowni delete qilib batch write qilish.
* **Batch Processing:** Ma'lumotlarni kichik paketlarda (chunks) qayta ishlash orqali oqim barqarorligini ta'minlash.

### 2.3 Minimalist Web UI
* Konnektorlarni yaratish, tahrirlash va o'chirish uchun sodda dashboard.
* Sinxronizatsiya jarayonining joriy holati (Success/Fail) va statistikasi.
* Tizim loglarini (Sync history) ko'rish imkoniyati.

---

## 3. Non-Functional Requirements
* **Lightweight Storage:** Konfiguratsiya bazasi sifatida SQLite ishlatish (minimal disk va RAM footprint).
* **Persistence:** SQLite ishlatish orqali tizim qayta ishga tushganda ishlarni to'xtagan joyidan (checkpoint) davom ettirishi.
* **Scalability:** Bir vaqtning o'zida bir nechta MongoDB collection'larini parallel ravishda ClickHouse'ga sinxron qila olish mantiqi.

---

## 4. System Objects

### 4.1 Connection Table (SQLite)
| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | INTEGER | Unique ID |
| `name` | TEXT | Konnektor nomi |
| `source_uri` | TEXT | MongoDB ulanish manzili |
| `dest_uri` | TEXT | ClickHouse ulanish manzili |
| `sync_status` | TEXT | `active`, `paused`, `error` |
| `last_run` | DATETIME | Oxirgi muvaffaqiyatli sync vaqti |

### 4.2 Sync Job
* **Extractor:** MongoDB cursor orqali ma'lumotlarni filtrlab o'qiydi.
* **Transformer:** JSON field-larni tekislaydi (Flattening) va ClickHouse tiplariga (String, Int32, DateTime) moslashtiradi.
* **Loader:** ClickHouse driver orqali batch `INSERT` query'larini bajaradi.

---

## 5. Telemetry & Monitoring
* **Metrics:** Sinxronizatsiya ko'rsatkichlari (ko'chirilgan qatorlar soni, ketgan vaqt).
* **Logs:** SQLite ichida saqlanuvchi tizim loglari (oxirgi 100-500 ta operatsiya).
* **Alerting:** Ketma-ket xatoliklar yuz berganda statusni avtomatik yangilash.

---

## 6. Configuration Example
Tizim saqlash mantiqi:
```json
{
  "engine": "Zenith-V1",
  "metadata_storage": {
    "type": "sqlite3",
    "path": "./zenith_metadata.db"
  },
  "processing": {
    "batch_size": 1000,
    "max_parallel_jobs": 4
  }
}