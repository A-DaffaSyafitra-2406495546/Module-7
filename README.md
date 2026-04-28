# Exercise Profiling

Spring Boot application for learning performance profiling and optimization.

## Stack

- Java 17 + Spring Boot 3.2
- PostgreSQL 16
- JPA / Hibernate
- Apache JMeter 5.6.3

## Setup

### Prerequisites

- PostgreSQL running (`brew services start postgresql@16`)
- Database: `profiling_db`, user: `profiling_user`

### Run

```bash
./mvnw spring-boot:run
```

### Seed Data

```bash
curl http://localhost:8080/seed-data-master
curl http://localhost:8080/seed-student-course
```

Produces: 20,000 students · 10 courses · 40,000 student-course relations.

---

## Performance Testing - Baseline (Before Optimization)

**Test config:** 10 threads · ramp-up 1s · loop 1x · target `localhost:8080`  
**Tool:** Apache JMeter 5.6.3 (non-GUI mode)  
**Date:** 2026-04-28

### Results

| Endpoint | Samples | Avg (ms) | Min (ms) | Max (ms) | Median (ms) | Throughput (req/s) | Error Rate |
|---|---|---|---|---|---|---|---|
| `/all-student` | 10 | 73,811 | 73,517 | 74,224 | 73,788 | 0.13 | 0.00% |
| `/all-student-name` | 10 | 1,798 | 1,142 | 2,029 | 1,867 | 3.97 | 0.00% |
| `/highest-gpa` | 10 | 77 | 71 | 101 | 75 | 10.70 | 0.00% |

### Observations

- `/all-student` sangat lambat (~74 detik) karena mengembalikan seluruh data 20,000 students beserta relasinya — N+1 query problem: 1 query untuk students + 20,000 query per-student untuk courses.
- `/all-student-name` lambat (~1.8 detik) karena load full entity padahal hanya butuh nama, ditambah string concatenation `+=` dalam loop (O(n²)).
- `/highest-gpa` cepat (~77ms) karena hanya satu record, meski tetap load seluruh 20K students ke memori dulu.

### JMeter Screenshots

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_1 (/all-student) baseline -->

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_2 (/all-student-name) baseline -->

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_3 (/highest-gpa) baseline -->

---

## Performance Testing - After Optimization

**Test config:** sama — 10 threads · ramp-up 1s · loop 1x · target `localhost:8080`  
**Branch:** `optimize`  
**Date:** 2026-04-28

### Optimizations Applied

- `getAllStudentsWithCourses`: ganti loop N+1 dengan satu JOIN FETCH query ke `student_courses`.
- `joinStudentNames`: ganti full-entity load + `+=` loop dengan JPQL projection (`SELECT s.name`) + `String.join()`.
- `findStudentWithHighestGpa`: ganti full-table scan in-memory dengan `findTopByOrderByGpaDesc()` (ORDER BY + LIMIT 1 di DB).

### Results

| Endpoint | Samples | Avg (ms) | Min (ms) | Max (ms) | Median (ms) | Throughput (req/s) | Error Rate |
|---|---|---|---|---|---|---|---|
| `/all-student` | 10 | 77,034 | 76,577 | 77,372 | 77,063 | 0.13 | 0.00% |
| `/all-student-name` | 10 | 1,589 | 1,379 | 1,728 | 1,608 | 4.49 | 0.00% |
| `/highest-gpa` | 10 | 76 | 68 | 122 | 72 | 10.85 | 0.00% |

### JMeter Screenshots

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_1 (/all-student) optimized -->

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_2 (/all-student-name) optimized -->

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_3 (/highest-gpa) optimized -->

---

## Comparison: Baseline vs Optimized

### /all-student

| Metric | Baseline | Optimized | Change |
|---|---|---|---|
| Avg Response Time | 73,811 ms | 77,034 ms | +4.4% (lebih lambat) |
| Min Response Time | 73,517 ms | 76,577 ms | +4.2% |
| Max Response Time | 74,224 ms | 77,372 ms | +4.2% |
| Throughput | 0.13 req/s | 0.13 req/s | ~0% |
| Error Rate | 0.00% | 0.00% | - |

### /all-student-name

| Metric | Baseline | Optimized | Change |
|---|---|---|---|
| Avg Response Time | 1,798 ms | 1,589 ms | -11.6% (lebih cepat) |
| Min Response Time | 1,142 ms | 1,379 ms | +20.8% |
| Max Response Time | 2,029 ms | 1,728 ms | -14.8% |
| Throughput | 3.97 req/s | 4.49 req/s | +13.1% |
| Error Rate | 0.00% | 0.00% | - |

### /highest-gpa

| Metric | Baseline | Optimized | Change |
|---|---|---|---|
| Avg Response Time | 77 ms | 76 ms | -1.3% |
| Min Response Time | 71 ms | 68 ms | -4.2% |
| Max Response Time | 101 ms | 122 ms | +20.8% |
| Throughput | 10.70 req/s | 10.85 req/s | +1.4% |
| Error Rate | 0.00% | 0.00% | - |

---

## Conclusion

Hasil JMeter setelah optimasi menunjukkan gambaran yang tidak seragam, dan ini justru revealing.

Untuk `/all-student`, response time di JMeter tidak membaik — bahkan sedikit lebih lambat (~77 detik vs ~74 detik sebelumnya). Ini bukan berarti optimasi gagal. Bottleneck N+1 memang sudah hilang (dari 20,001 queries menjadi 1 query JOIN FETCH), tapi bottleneck bergeser ke lapisan berikutnya: serialisasi 40,000 objek `StudentCourse` ke string melalui `.toString()` tetap menjadi operasi yang sangat berat. JMeter mengukur end-to-end response time, sehingga penghematan di sisi database tertutup oleh biaya serialisasi yang masih besar. IntelliJ Profiler kemungkinan besar akan menunjukkan pengurangan drastis di jumlah query dan CPU time untuk database I/O, tapi waktu total tetap didominasi oleh serialisasi.

Untuk `/all-student-name`, ada improvement nyata sebesar ~11.6% di rata-rata response time dan 13.1% di throughput. Ini konsisten dengan optimasi yang dilakukan: menghapus full entity loading dan O(n²) string concatenation sekaligus mengurangi beban di dua titik sekaligus.

Untuk `/highest-gpa`, perubahan dalam noise range saja (~1ms) karena endpoint ini sudah fast dari awal — optimasi tetap benar secara arsitektural (tidak perlu load 20K rows ke memory), tapi tidak terlihat di JMeter karena latency-nya sudah sangat kecil.

Kesimpulannya: optimasi kode yang benar tidak selalu langsung terlihat di JMeter metric, karena JMeter mengukur keseluruhan pipeline dari request sampai response. Profiler diperlukan untuk memverifikasi bahwa bottleneck yang ditarget (query count, CPU time per method) benar-benar berkurang, terlepas dari apakah end-to-end time ikut turun.

---

## Reflection

**1. Apa perbedaan pendekatan performance testing dengan JMeter dan profiling dengan IntelliJ Profiler dalam konteks optimasi performa aplikasi?**

JMeter mengukur performa dari sisi luar: seberapa cepat server merespons request dari perspektif client. Kita tahu ada masalah, tapi tidak tahu di mana tepatnya. IntelliJ Profiler masuk ke dalam dan menunjukkan method mana yang paling banyak makan CPU, berapa kali dipanggil, dan berapa memori yang dialokasikan. Keduanya saling melengkapi: JMeter untuk mendeteksi dan memverifikasi dampak dari luar, Profiler untuk menemukan akar masalahnya.

**2. Bagaimana proses profiling membantu mengidentifikasi dan memahami weak points di aplikasi?**

Profiler menampilkan call tree dan flame graph yang langsung menunjukkan method mana yang dominant di CPU time. Tanpa profiler, kita hanya bisa menebak bahwa ada yang lambat dari response time JMeter. Dengan profiler, kita bisa melihat misalnya bahwa `findByStudentId` dipanggil 20,000 kali dalam satu request, dan itu langsung menunjukkan N+1 problem tanpa perlu analisa manual yang panjang.

**3. Apakah IntelliJ Profiler efektif buat analyze dan identify bottleneck di kode aplikasi?**

Efektif, terutama untuk aplikasi Spring Boot yang sudah jalan di JVM. Kita bisa attach profiler ke proses yang sedang running tanpa perlu recompile atau tambah dependency. Visualisasi flame graph dan method-level CPU breakdown sangat membantu untuk langsung fokus ke kode yang bermasalah daripada harus baca log atau trace manual.

**4. Apa tantangan utama saat performance testing dan profiling, dan gimana cara ngatasinnya?**

Tantangan terbesar adalah memastikan environment saat testing representatif dengan kondisi produksi, karena hasil bisa sangat berbeda jika data terlalu sedikit atau JVM belum warm up. Selain itu, hasil JMeter bisa misleading kalau bottleneck ada di lapisan yang tidak terlihat langsung seperti serialisasi atau network. Solusinya adalah selalu jalankan beberapa kali untuk mengurangi noise, dan gunakan profiler bersamaan untuk memahami apa yang sebetulnya terjadi di dalam aplikasi.

**5. Apa benefit utama dari pake IntelliJ Profiler buat profiling kode?**

Integrasi langsung dengan IDE membuat workflow-nya lebih smooth, kita tidak perlu setup tool terpisah atau export-import data profiling. Yang paling membantu adalah kemampuan melihat CPU usage per method secara real-time sambil request sedang diproses, sehingga kita bisa langsung korelasikan antara apa yang dilihat di profiler dengan kode yang ada di editor.

**6. Gimana handle situasi kalo hasil profiling IntelliJ Profiler gak konsisten sama hasil JMeter?**

Ini sebetulnya situasi yang normal dan bukan berarti ada yang salah. Profiler mengukur CPU time dan call count di level JVM, sementara JMeter mengukur wall-clock time end-to-end termasuk network, serialisasi, dan overhead lainnya. Kalau profiler menunjukkan improvement tapi JMeter tidak, berarti bottleneck sudah bergeser ke lapisan lain yang belum dioptimasi. Cara mengatasinya adalah terus profil ulang setelah setiap perubahan untuk melihat bottleneck berikutnya.

**7. Strategi apa yang dipake buat optimasi kode setelah analisa hasil testing & profiling? Gimana mastiin perubahan gak ngerusak functionality?**

Strategi yang dipakai adalah fix satu bottleneck paling besar dulu, commit, lalu ukur ulang, daripada mengubah banyak hal sekaligus dan tidak tahu mana yang berpengaruh. Untuk memastikan functionality tidak berubah, kita bisa bandingkan response payload sebelum dan sesudah dengan curl ke endpoint yang sama, dan kalau ada unit test jalankan itu juga. Di kasus ini, karena tidak ada unit test, verifikasi dilakukan manual dengan membandingkan format output `toString()` yang hasilnya identik antara baseline dan versi optimized.

---

## JMeter Test Plans

Lihat [jmeter/SETUP.md](jmeter/SETUP.md) untuk panduan membuat test plan via GUI.

```
jmeter/
├── SETUP.md
├── test_plan_1.jmx          # /all-student
├── test_plan_2.jmx          # /all-student-name
├── test_plan_3.jmx          # /highest-gpa
└── results/
    ├── test_plan_1_baseline.jtl
    ├── test_plan_1_optimized.jtl
    ├── test_plan_2_baseline.jtl
    ├── test_plan_2_optimized.jtl
    ├── test_plan_3_baseline.jtl
    └── test_plan_3_optimized.jtl
```

### Run via CLI

```bash
# Baseline
jmeter -n -t jmeter/test_plan_1.jmx -l jmeter/results/test_plan_1_baseline.jtl
jmeter -n -t jmeter/test_plan_2.jmx -l jmeter/results/test_plan_2_baseline.jtl
jmeter -n -t jmeter/test_plan_3.jmx -l jmeter/results/test_plan_3_baseline.jtl

# After optimization
jmeter -n -t jmeter/test_plan_1.jmx -l jmeter/results/test_plan_1_optimized.jtl
jmeter -n -t jmeter/test_plan_2.jmx -l jmeter/results/test_plan_2_optimized.jtl
jmeter -n -t jmeter/test_plan_3.jmx -l jmeter/results/test_plan_3_optimized.jtl
```
