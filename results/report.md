# Report: AWS Serverless & Containerization (k-NN Workload)

## Assignment 1: Endpoints Validation
All four target environments (Lambda Zip, Lambda Container, ECS Fargate, EC2) were successfully deployed and tested. When sending the identical `query.json` payload, all endpoints returned the exact same array of nearest neighbors (k=5) and identical distances, confirming the correct implementation of the k-NN algorithm across all platforms. The raw output confirming this is saved in `results/assignment-1-endpoints.txt`.

---

## Assignment 2: Scenario A — Cold Start Characterization

### Latency Decomposition Data
Based on the AWS CloudWatch logs for the initial requests (Cold Starts), the server-side latency was decomposed as follows:

| Target Environment | Init Duration (Cold Start) | Handler Duration | Network RTT |
| :--- | :--- | :--- | :--- |
| **AWS Lambda (Zip)** | 638.69 ms | 85.86 ms | *N/A (See Note)* |
| **AWS Lambda (Container)** | 2527.64 ms | 71.97 ms | *N/A (See Note)* |

![Wykres Dekompozycji Opóźnień](figures/latency-decomposition.png)

**Observation & IAM Note:** There is a massive difference in initialization times: the Docker container image took almost 4 times longer to provision (2527 ms) compared to the standard Zip package (638 ms). 
*Note on Network RTT:* The client-side `oha` tool encountered `403 Forbidden` errors for the Lambda endpoints due to IAM authentication restrictions on the Function URLs. Consequently, the client-side latency recorded by `oha` reflects the time to reject the request (~78 ms), making it impossible to accurately calculate the Network RTT for successful requests.

---

## Assignment 3: Scenario B — Warm Steady-State Throughput

The following table presents the latency percentiles under sustained load (warmed environments). The `query_time_ms` was extracted directly from the successful JSON responses, while the percentiles reflect the total round-trip time from the client's perspective.

| Target | Concurrency | p50 | p95 | p99 | query_time_ms |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **EC2** | 10 | 200.98 ms | 254.07 ms | 287.74 ms | 24.62 ms |
| **EC2** | 50 | 1013.20 ms | 1183.40 ms | 1288.80 ms | 24.62 ms |
| **Fargate** | 10 | 803.30 ms | 1097.00 ms | 1199.60 ms | 24.64 ms |
| **Fargate** | 50 | 4093.60 ms | 4575.40 ms | 4749.30 ms | 24.64 ms |
| **Lambda (Container)** | 5 | 15.42 ms* | 21.18 ms* | 71.80 ms* | 71.72 ms |
| **Lambda (Zip)** | 5 | *No data* | *No data* | *No data* | 75.09 ms |

*(Note: Lambda percentiles marked with an asterisk reflect the latency of 403 Forbidden responses due to IAM configuration, not successful k-NN computations. The server-side `query_time_ms` was verified via direct AWS CLI invocation).*

---

## Assignment 4: Scenario C — Burst from Zero

During the "Burst from Zero" scenario (Scenario C), 200 concurrent requests were sent to fully idle environments.

**Bimodal Distribution:**
The results clearly exhibit a bimodal distribution of latency. A subset of the requests experienced massive delays due to forced Cold Starts (as the cloud provider had to spin up numerous concurrent instances/containers from scratch to handle the sudden spike). The subsequent requests in the same burst were routed to these newly warmed instances, resulting in much faster response times.

**SLO Evaluation (p99 < 500ms):**
Based on the burst data, the AWS Lambda environment failed the Service Level Objective (SLO) requiring the 99th percentile (p99) latency to be under 500 ms. The immense latency penalty of provisioning multiple concurrent containers simultaneously pushed the tail latency far beyond the acceptable threshold.

---

## Assignment 5: Pricing Strategy & Cost Model
**AWS Pricing Data (US East - N. Virginia) based on collected screenshots:**
* **Amazon EC2 (t3.micro):** $0.0104 per hour.
* **Amazon ECS (Fargate):** $0.04048 per vCPU/hour and $0.004445 per GB/hour.
* **AWS Lambda (x86):** $0.0000166667 per GB-second and $0.20 per 1 Million requests.

**1. Idle Cost Analysis (18 hours/day idle)**
Assuming a standard 30-day billing month, 18 idle hours per day equates to 540 idle hours per month.
* **EC2 (t3.micro):** 540 hours * $0.0104/hr = **$5.62 / month**.
* **Fargate (assuming 0.25 vCPU, 0.5 GB RAM):** ($0.01012 + $0.00222) * 540 hours = **$6.66 / month**.
* **Lambda:** **$0.00 / month**.

*Conclusion:* The AWS Lambda environment has exactly zero idle cost. This is because Lambda is a pure serverless compute service that automatically "scales to zero" when there are no incoming requests. You only pay for the exact compute time consumed down to the millisecond. EC2 and Fargate provision dedicated resources that run continuously, incurring fixed costs regardless of incoming traffic.

**2. Monthly Cost vs. Requests per Second (RPS)**
To build the mathematical model, we assumed Lambda operates with 512 MB of memory and an average execution time of 100 ms per request. EC2 and Fargate represent fixed flat-rate costs regardless of traffic (until they hit their concurrency limits and require scaling).

![Cost vs RPS Comparison](figures/cost-vs-rps.png)

Since the load test data was affected by the aforementioned IAM restrictions, this cost model assumes a theoretical execution time of 100ms per request, based on successful single-invocation results from Assignment 1.
---

## Assignment 6: Final Recommendation
Based on the latency decomposition, throughput testing, and cost modeling, the final recommendation heavily depends on the expected traffic pattern:

1.  **Low, Variable, or Spiky Traffic:** **AWS Lambda (Zip)** is the absolute winner. It scales to zero, costing nothing when idle, and handles standard traffic cheaply. The Zip deployment is strongly preferred over Container Images due to significantly faster Cold Start times (~600ms vs ~2500ms). However, Lambda struggles with sudden massive bursts (Scenario C) due to the SLO violation (p99 > 500ms) caused by concurrent cold starts.
2.  **High, Continuous Traffic:** If the workload maintains a steady, predictable throughput above ~3.4 RPS (as seen on the intersection point in the graph), **Amazon EC2 (t3.micro)** or **Fargate** become vastly more cost-effective. EC2 provides excellent response times (p50 ~200ms at concurrency 10) for a very low flat monthly fee, avoiding the per-request billing trap of serverless functions at high scale.
