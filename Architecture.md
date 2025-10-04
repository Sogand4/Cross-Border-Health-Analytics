[Mobile App]


[API Gateway]


[AWS Lambda Router]


[S3 Buckets (CA/EU/BR, Raw → Aggregated)]


[AWS Glue Job (Nightly Transform)]


[S3 Aggregated Zone]


[Athena / QuickSight Dashboards]


---

Design Choice Explanations:

### API Gateway
- Exposes HTTPS `/upload` endpoint for mobile app uploads.
- Triggers Lambda; no caching used (uploads are unique).
- Regional pricing:
  - Canada (Central): $3.50 / 1M requests
  - EU (Frankfurt): $3.70 / 1M requests
  - Brazil (São Paulo): $4.25 / 1M requests
- Example: 10 M requests each → ≈ $115 USD/month total.
- Reference: [AWS API Gateway Pricing](https://aws.amazon.com/api-gateway/pricing/)


---

### AWS Lambda
- Processes each `/upload` request from API Gateway, validates metadata, and routes to the correct S3 bucket.
- Config: 128 MB memory, ~100 ms duration per invocation.
- Regional pricing (x86):
  - Canada (Central): $0.20 / 1 M requests + $0.00001667 per GB-second  
  - EU (Frankfurt): $0.20 / 1 M requests + $0.00001667 per GB-second  
  - Brazil (São Paulo): $0.20 / 1 M requests + $0.00001667 per GB-second
- Example: 10 M requests / region → ≈ $4 per region (≈ $12 total for 3 regions).
- Reference: [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)

