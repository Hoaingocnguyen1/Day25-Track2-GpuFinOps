# NimbusAI — GPU Cost Optimization Report

**Period:** monthly  
**Baseline spend:** $27,133  
**Optimized spend:** $14,626  
**Projected savings:** $12,507  (**46%**)

## Savings by lever

| Lever | Savings (USD) |
|---|---|
| Inference (cascade/cache/batch) | $1,212 |
| Purchasing (spot/reserved) | $10,040 |
| Right-size util-lies | $655 |
| Kill idle GPUs | $600 |

## Sustainability

- Energy per query: 0.24 Wh
- Carbon per query: 0.091 gCO2e
- Cheapest+cleanest region: europe-north1

## 1. Executive summary

NimbusAI can reduce the modeled monthly GPU bill from **$27,133 to $14,626**, saving **$12,507/month (46.1%)**. The largest opportunity is purchasing optimization, which contributes **$10,040 (80.3%)** of projected savings.

For inference, cascade routing, prompt caching, and batching reduce unit cost from **$6.488 to $1.126/1M tokens**, an **82.6%** improvement. This is the preferred operating KPI because it measures useful output rather than only the hourly rental price.

## 2. Efficiency audit (M1)

| GPU | Type | GPU-Util | MFU | MBU | Finding |
|---|---|---:|---:|---:|---|
| gpu-h100-4 | H100 | 98.2% | 0.194 | 0.207 | GPU-Util lie |
| gpu-a10g-1 | A10G | 96.9% | 0.268 | 0.302 | GPU-Util lie |

GPU-Util only indicates that the device was busy during the sampling window. It does not show how much peak compute was converted into model work. Memory stalls, synchronization, I/O waits, small kernels, or kernel launch overhead can therefore produce high GPU-Util with low MFU. Paying for an H100 under this condition buys capacity that the workload does not use.

Idle intervals waste **$20.00/day**, or **$600/month**. Shutting down idle GPUs and right-sizing the two util-lie devices provide a modeled additional **$1,255/month**.

## 3. Inference economics (M2)

The sample contains **7,533,027 tokens across 2,400 requests**. Daily inference cost falls from **$48.87** to **$8.48**, equivalent to **$1,212/month** in savings.

The three levers are multiplicative: cascade sends eligible traffic to a smaller model, caching discounts reused input, and batch mode halves the eligible request cost. A fully cached batched input costs 5% of its naive input price (50% batch factor × 10% cached-input factor). Batch mode should remain limited to latency-tolerant traffic.

## 4. Purchasing strategy (M3)

At a 45% reserved discount, break-even utilization is **55%**. The policy moves interruptible workloads to spot and steady workloads above this duty cycle to reserved capacity. Purchasing cost falls from **$25,667 to $15,627/month** (**39.1%**).

| Workload | GPU | Recommended tier | On-demand | Optimized |
|---|---|---|---:|---:|
| job-train-llm | H100 | spot | $12,000 | $7,596 |
| job-train-embed | A100 | spot | $2,148 | $1,393 |
| job-finetune | H100 | spot | $900 | $570 |
| job-infer-chat | A10G | reserved | $4,320 | $2,592 |
| job-infer-rag | A100 | reserved | $3,866 | $2,160 |
| job-infer-search | L4 | reserved | $1,728 | $972 |
| job-dev-sandbox | A10G | spot | $480 | $203 |
| job-batch-eval | H100 | spot | $225 | $142 |

Spot workloads require checkpointing and interruption monitoring. Reserved capacity should only be committed after validating that the observed duty cycle is stable for the intended contract period.

## 5. Allocation and governance (M4)

Tag coverage is **92%**, above the 80% chargeback gate; chargeback readiness is therefore **True**. The FOCUS export normalizes billing fields and team/project tags so the same allocation workflow can be used across cloud providers.

| Team | Optimized inference cost/day | Share |
|---|---:|---:|
| assistant | $2.59 | 30.6% |
| search | $2.49 | 29.4% |
| eval | $1.79 | 21.1% |
| rag | $1.60 | 18.9% |

## 6. Sustainability

A representative 800-token query consumes **0.24 Wh** and emits **0.091 gCO2e** in us-east-1. Among the modeled regions, **europe-north1** has the lowest carbon intensity. Region movement must still account for latency, data residency, and egress costs.

## 7. Prioritized action plan

1. Apply the spot/reserved policy first, with checkpointing and utilization alerts; it represents about 80% of modeled savings.
2. Deploy cascade, caching, and batch routing while tracking $/1M tokens and latency/SLOs by route.
3. Shut down idle capacity and profile util-lie GPUs before right-sizing them; use MFU/MBU rather than GPU-Util alone.
4. Start showback immediately and move to chargeback only after maintaining tag coverage above the threshold.

## 8. Assumptions and limitations

- Results use deterministic synthetic data and June 2026 price snapshots.
- Lever savings are treated as additive; production validation must remove overlap between purchasing, right-sizing, and idle-capacity savings.
- Quality, latency, interruption frequency, cache storage cost, and migration cost are not included in the headline projection.
- No optional ‘Your Turn’ extension has been implemented in this baseline report.

## 9. Validation

- `python verify.py`: **11/11 checks passed**
- `pytest -q`: **15/15 tests passed**

_Figures are June-2026 as-of snapshots; re-baseline before acting._