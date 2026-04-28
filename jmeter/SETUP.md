# JMeter Test Plan Setup Guide

Target app: `http://localhost:8080`  
Tool: Apache JMeter 5.6.3

---

## Persyaratan

- Spring Boot app harus jalan dulu: `./mvnw spring-boot:run`
- JMeter terinstall: `brew install jmeter`
- Buka JMeter GUI: `jmeter` (tanpa argumen)

---

## Membuat Test Plan via GUI

Ulangi langkah-langkah berikut untuk **masing-masing** dari 3 test plan.

---

### Test Plan 1 — `/all-student`

#### Step 1: Buat Test Plan baru
1. Buka JMeter → akan ada **Test Plan** kosong di panel kiri.
2. Klik nama "Test Plan" di panel kiri, ubah **Name** jadi: `Test Plan 1 - All Student`

#### Step 2: Tambah Thread Group
1. Klik kanan **Test Plan** → **Add** → **Threads (Users)** → **Thread Group**
2. Isi konfigurasi:
   - **Name**: `Thread Group`
   - **Number of Threads (users)**: `10`
   - **Ramp-up period (seconds)**: `1`
   - **Loop Count**: `1` (pastikan radio button "Loop Count" dipilih, bukan "Infinite")

#### Step 3: Tambah HTTP Request
1. Klik kanan **Thread Group** → **Add** → **Sampler** → **HTTP Request**
2. Isi konfigurasi:
   - **Name**: `HTTP Request - /all-student`
   - **Protocol**: `http`
   - **Server Name or IP**: `localhost`
   - **Port Number**: `8080`
   - **HTTP Request** (method): `GET`
   - **Path**: `/all-student`

#### Step 4: Tambah 4 Listeners
Klik kanan **Thread Group** → **Add** → **Listener**, tambahkan satu per satu:

| # | Menu Path | Name yang dipakai |
|---|-----------|-------------------|
| 1 | Listener → View Results Tree | `View Results Tree` |
| 2 | Listener → View Results in Table | `View Results in Table` |
| 3 | Listener → Summary Report | `Summary Report` |
| 4 | Listener → Graph Results | `Graph Results` |

#### Step 5: Save
- **File** → **Save Test Plan As** → simpan ke `jmeter/test_plan_1.jmx`

---

### Test Plan 2 — `/all-student-name`

Ulangi semua langkah di atas dengan perubahan berikut:
- **Test Plan Name**: `Test Plan 2 - All Student Name`
- **HTTP Request Name**: `HTTP Request - /all-student-name`
- **Path**: `/all-student-name`
- Simpan ke: `jmeter/test_plan_2.jmx`

---

### Test Plan 3 — `/highest-gpa`

Ulangi semua langkah di atas dengan perubahan berikut:
- **Test Plan Name**: `Test Plan 3 - Highest GPA`
- **HTTP Request Name**: `HTTP Request - /highest-gpa`
- **Path**: `/highest-gpa`
- Simpan ke: `jmeter/test_plan_3.jmx`

---

## Struktur File yang Diharapkan

```
jmeter/
├── SETUP.md
├── test_plan_1.jmx
├── test_plan_2.jmx
├── test_plan_3.jmx
└── results/
    ├── test_plan_1_baseline.jtl
    ├── test_plan_2_baseline.jtl
    └── test_plan_3_baseline.jtl
```

---

## Menjalankan Test via CLI (setelah .jmx tersimpan)

```bash
jmeter -n -t jmeter/test_plan_1.jmx -l jmeter/results/test_plan_1_baseline.jtl
jmeter -n -t jmeter/test_plan_2.jmx -l jmeter/results/test_plan_2_baseline.jtl
jmeter -n -t jmeter/test_plan_3.jmx -l jmeter/results/test_plan_3_baseline.jtl
```

Flag `-n` = non-GUI (headless), `-t` = test plan, `-l` = log/output file.

---

## Tips

- Kalau listener di .jmx punya atribut `filename`, JMeter akan nulis output ke file tersebut secara otomatis. Kosongkan field "Filename" di listener saat bikin via GUI supaya tidak bentrok dengan `-l` flag.
- Untuk buka hasil `.jtl` di GUI: buka listener mana saja → klik tombol folder → pilih file `.jtl`.
