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

- `/all-student` sangat lambat (~74 detik) karena mengembalikan seluruh data 20,000 students beserta relasinya — kemungkinan N+1 query problem atau lazy loading yang tidak efisien.
- `/all-student-name` lebih cepat (~1.8 detik) karena hanya mengembalikan nama saja, tapi masih terasa lambat untuk data 20,000 rows.
- `/highest-gpa` sangat cepat (~77ms) karena hanya mengembalikan satu record.

### JMeter Screenshots

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_1 (/all-student) -->

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_2 (/all-student-name) -->

<!-- SCREENSHOT: JMeter Summary Report untuk test_plan_3 (/highest-gpa) -->

<!-- SCREENSHOT: JMeter Graph Results atau View Results in Table (opsional, salah satu endpoint) -->

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
    ├── test_plan_2_baseline.jtl
    └── test_plan_3_baseline.jtl
```

### Run via CLI

```bash
jmeter -n -t jmeter/test_plan_1.jmx -l jmeter/results/test_plan_1_baseline.jtl
jmeter -n -t jmeter/test_plan_2.jmx -l jmeter/results/test_plan_2_baseline.jtl
jmeter -n -t jmeter/test_plan_3.jmx -l jmeter/results/test_plan_3_baseline.jtl
```
