# Insight Memo

**Project:** Patient Symptom Ingestion & Analytics Pipeline

---

### 1. Insight: Batch Uploads Dramatically Reduce Cost and Latency
**Evidence:** Back-of-envelope cost calculation; 2 TB nightly ingestion requires ~2M PUT requests per region. Batch uploads reduce Lambda invocations by ~70%, keeping monthly compute under $10 per region. CloudWatch logs confirm 100–200ms average validation per batch.  

**Why it matters:** Lower Lambda invocations reduce serverless compute costs and network overhead, enabling the system to scale to thousands of users without breaking the budget. It also ensures faster ingestion and less throttling during peak upload periods, improving patient experience.  

**Limits:** Assumes users consistently submit full daily batches; sporadic or partial uploads reduce efficiency. Peak-size payloads (>2 MB) may require more memory.  

**Next question:** Can we dynamically adjust batch size per user to optimize for both cost and latency?

---

### 2. Insight: Regional Bucket Separation Prevents Privacy Violations
**Evidence:** IAM and S3 replication policy review; no cross-region replication detected in audit logs. Test payloads tagged as “Indigenous” remained in `ca-central-1` throughout ETL process.  

**Why it matters:** Confirms compliance with jurisdictional privacy requirements (PIPEDA, GDPR, LGPD, OCAP®) and protects sensitive patient health data. This builds trust and reduces legal risk.  

**Limits:** Only tested on synthetic and limited real-world payloads. Misconfiguration or rogue replication could break assumptions.  

**Next question:** How can we automate continuous compliance validation across multiple regions?

---

### 3. Insight: Compression Vastly Decreases Storage and Query Costs
**Evidence:** Raw JSON data (~2 TB/night) is compressed 5× into Parquet format in S3. Storage cost estimates: uncompressed ~ $235/month per region → compressed ~ $47/month. Athena query costs reduced proportionally (90% reduction in scanned TB).

**Why it matters:** Compression drastically lowers both storage and query costs while preserving analytical value. This makes the solution economically sustainable for multi-region, large-scale patient data ingestion.  

**Limits:** Very high-cardinality or complex nested JSON may compress less efficiently; real-time query performance could be affected by decompression overhead.  

**Next question:** Can we safely increase compression ratio without losing downstream ETL or analytics fidelity?

---