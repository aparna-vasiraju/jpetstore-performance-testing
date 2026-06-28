## Live Monitoring Dashboard

Real-time performance monitoring implemented using 
JMeter → InfluxDB → Grafana pipeline.

![Grafana Dashboard](screenshots/grafana_live_dashboard.png)
# JPetStore Performance Test Suite

End-to-end performance test suite built with Apache JMeter for the JPetStore e-commerce application. Covers browsing and buying user flows with baseline, load, and stress test scenarios.

---

## Project Overview

JPetStore is a classic Java/J2EE e-commerce demo application. This project simulates realistic mixed user traffic — browsers exploring the catalog and buyers completing purchases — to evaluate system behaviour under increasing load.

**Application under test:** https://petstore.octoperf.com  
**Tool:** Apache JMeter 5.6.3  
**Test duration:** June 2026  

---

## Test Architecture

### Thread Groups

| Thread Group | Flow | Transactions |
|---|---|---|
| UTG - Browsing Flow | Anonymous users exploring the catalog | Home → Category → Product → Item |
| UTG - Buyers Flow | Authenticated users completing purchases | Home → Login → Category → Product → Add to Cart → Checkout → Confirm → Sign Out |

### Key Design Decisions

- **Parameterization:** Unique users per thread via CSV Data Set Config (Sharing Mode: Current thread group)
- **Correlation:** Dynamic tokens (`_sourcePage`, `__fp`) extracted per request using Regex Extractors — these tokens change on every response and are required for successful form submissions
- **Assertions:** Response Assertion on login step checks for `Welcome` in response body — silent login failures are caught immediately
- **Error handling:** Thread group configured with `Start Next Thread Loop` on sampler error — broken sessions are abandoned rather than continuing with null user context
- **Think time:** Constant Timer between transactions to simulate realistic user behaviour

---

## Test Data

```csv
username,password,categoryName,productId,workingItemId
perf_user1,perf123,FISH,FI-SW-01,EST-1
perf_user2,perf123,DOGS,K9-BD-01,EST-6
perf_user3,perf123,CATS,FL-DLH-02,EST-14
perf_user4,perf123,BIRDS,AV-CB-01,EST-18
perf_user5,perf123,REPTILES,RP-SN-01,EST-11
```

Each thread uses a unique user account and navigates a different product category — ensuring realistic mixed traffic rather than all threads hitting the same data.

---

## Test Scenarios

| Scenario | Users | Ramp Up | Hold | Purpose |
|----------|-------|---------|------|---------|
| Baseline | 1     | 1 sec   | 60 sec | Establish baseline response times |
| Load Test | 10   | 20 sec  | 120 sec | Validate performance under normal load |
| Stress Test | 25 | 30 sec  | 120 sec | Identify behaviour under peak load |

---

## Results Summary

### Response Times (ms)

| Transaction | Baseline Avg | Load Avg | Stress Avg | Baseline 90% | Load 90% | Stress 90% |
|-------------|--------------|----------|------------|--------------|----------|------------|
| Home        | 719          | 765      | 734        | 771          | 773      | 773        |
| Sign In Page | 180         | 180      | 181        | 194          | 194      | 194        |
| Submit Login | 536         | 536      | 540        | 574          | 576      | 577        |
| Category     | 178         | 178      | 178        | 191          | 191      | 192        |
| Product      | 178         | 177      | 179        | 192          | 191      | 192        |
| Add To Cart  | 182         | 182      | 183        | 196          | 196      | 196        |
| Checkout     | 180         | 180      | 181        | 194          | 195      | 194        |
| Confirm Order| 183         | 184      | 185        | 196          | 197      | 198        |
| Sign Out     | 535         | 535      | 538        | 574          | 575      | 577        |

### Error Rate

| Scenario | Users | Error % |
|----------|-------|---------|
| Baseline | 1     | 0.00%   |
| Load Test| 10    | 0.00%   |
| Stress Test| 25  | 0.00%   |

---

## Key Findings

**System is highly stable under load**
Response times showed less than 3% degradation when scaling from 1 user to 25 concurrent users. The home page response time moved from 719ms (baseline) to 734ms (stress) — a negligible increase indicating the application handles this load tier well.

**Authentication is not a bottleneck**
Login response time remained consistent at ~536ms across all load levels, suggesting the authentication layer scales independently of concurrent session count.

**Throughput scaled linearly**
Buyers Flow throughput increased from 1.5 req/sec (baseline) to 7.1 req/sec (stress) — close to linear scaling, indicating no queuing or contention issues at this load level.

**Correlation complexity**
JPetStore uses dynamic CSRF-style tokens (`_sourcePage`, `__fp`) that change on every response. Successfully implemented per-request extraction and replay using JMeter Regex Extractors. This was validated by running concurrent users — a common failure point where shared or stale tokens cause session collisions and server-side null constraint violations.

---

## Defect Found During Scripting

During concurrent user testing, a `500 Internal Server Error` was identified:

```
integrity constraint violation: NOT NULL check constraint
table: ORDERS column: USERID
```

Root cause analysis using JMeter Debug Sampler revealed that one thread was using invalid credentials (`AC_USER_1 / pass123`) — causing a silent login failure. The application returned HTTP 200 on the login page (no redirect), so JMeter proceeded with a session that had no authenticated user. When the order was placed, `USERID` was null, violating the database constraint.

Fix: Added Response Assertion on login to detect silent failures, configured thread group to abandon the iteration on assertion failure, and corrected test data to use valid registered credentials.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Apache JMeter 5.6.3 | Test scripting, execution, reporting |
| JMeter Plugins (Ultimate Thread Group) | Advanced thread scheduling |
| Regex Extractor | Dynamic token correlation |
| CSV Data Set Config | Test data parameterization |
| Aggregate Report | Results analysis |

---

## Repository Structure

```
jpetstore-performance-testing/
├── JMeterScript/
│   └── JPetStoreView Results Tree.jmx
├── TestData/
│   └── users.csv
├── Results/
│   ├── baseline_1user.csv
│   ├── loadtest_10users.csv
│   └── stress_25users.csv
├── Screenshots/
│   ├── baseline_aggregate.png
│   ├── loadtest_aggregate.png
│   └── stress_aggregate.png
└── README.md
```

---

## Author

**Aparna Vasiraju**  
Senior Performance Engineer | AWS Certified AI Practitioner  
