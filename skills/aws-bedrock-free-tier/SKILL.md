---
name: aws-bedrock-free-tier
version: 1.0
description: "Use when user wants free AI on AWS Bedrock. Models gated."
tags: [aws, bedrock, free-tier, ai-api, image-generation, polly, llama, claude, nova]
related_skills: [aws-free-ai-access]
---

# AWS Bedrock Free-Tier Reality

Amazon Bedrock is on AWS Free Tier, but almost no models are immediately invokable on a fresh account. Set expectations BEFORE promising image gen or Claude access.

## TL;DR for a fresh AWS free-tier account

| Service | Free? | Immediate invoke? |
|---|---|---|
| **Polly** (TTS) | 5M chars/mo forever | ✅ Yes |
| **S3** (storage) | 5 GB forever | ✅ Yes |
| **Lambda** | 1M invokes/mo forever | ✅ Yes |
| **Bedrock SDK** | yes | ✅ Client creates, ❌ model invoke gated |
| **Nova Canvas** (image) | yes | ❌ Needs Marketplace settle ~15min |
| **Nova Pro/Lite/Micro** (text) | yes | ❌ Same Marketplace gate |
| **Claude (Anthropic)** | yes | ❌ Marketplace + use-case form required |
| **Llama (Meta), Mistral** | yes | ❌ Marketplace auto-subscribe settle |
| **Titan Image Generator v1/v2** | EOL | ❌ `ResourceNotFoundException` — never works |
| **Stability SDXL v0/v1** | EOL | ❌ Same |
| **Claude 3.5/3.7 Sonnet (old IDs)** | EOL | ❌ Same |

## Error code → cause → fix

| Error | Cause | Fix |
|---|---|---|
| `ValidationException: Operation not allowed` on Amazon models | Marketplace auto-subscribe not settled, or region lacks the model | Wait 15–20 min; verify region is `us-east-1` or `us-west-2`; Console → Bedrock → Model access |
| `Operation not allowed` on Claude specifically | Use case form not submitted | Console → Model catalog → Claude → **Submit use case details** (mandatory first-time form). Or `PutUseCaseForModelAccess` API |
| `ResourceNotFoundException: model has reached the end of its life` | Model deprecated by AWS | Swap to a current model. Titan v1/v2, Stability SDXL v0/v1, Claude 3.5/3.7 Sonnet (old IDs), Cohere Command R/R+ are all EOL |
| `curl: (6) Could not resolve host: bedrock-mantle...` | Mantle endpoint not publicly launched | Do not depend on Mantle bearer tokens today. `boto3` also needs `MantleConfig` (botocore ≥ ~1.43) |
| `Unknown service: 'aws-marketplace'` in boto3 | Wrong client name | Correct names: `marketplacecommerceanalytics` or `marketplace-agreement` |

## Model IDs that work (2026-10 verified live list check)

Tested against `list_foundation_models()` on us-east-1. Cross-region inference count was 0 — models are region-scoped.

**Currently invokable after settle**:
- `amazon.nova-pro-v1:0`, `amazon.nova-lite-v1:0`, `amazon.nova-micro-v1:0`
- `amazon.nova-2-lite-v1:0`, `amazon.nova-2-lite-v1:0:256k`
- `amazon.nova-2-sonic`, `amazon.nova-2-5-sonic`
- `amazon.nova-canvas-v1:0` (image)
- `anthropic.claude-sonnet-4-20250514-v1:0`, `anthropic.claude-haiku-4-5-20251001-v1:0`, `anthropic.claude-sonnet-4-5-20250929-v1:0`
- `meta.llama3-3-70b-instruct-v1:0`, `meta.llama3-1-70b-instruct-v1:0`, `meta.llama3-1-8b-instruct-v1:0`
- `mistral.mistral-large-2411-v1:0`, `mistral.mistral-small-2402-v1:0`

**Never invoke (dead)**: `amazon.titan-image-generator-v1`, `amazon.titan-image-generator-v2:0`, `stability.stable-diffusion-xl-v0/v1`, `anthropic.claude-3-5-sonnet-20241022-v2:0`, `anthropic.claude-3-7-sonnet-20250219-v1:0`, `cohere.command-r-v1:0`, `cohere.command-r-plus-v1:0`.

## Mantle endpoint (OpenAI-compatible)

`bedrock-mantle.<region>.amazonaws.com` currently returns **NXDOMAIN** from Google DNS (Status 3). Mantle bearer tokens (`ABSK...`, up to 365d) are issued by the Console and are valid, but the endpoint is not publicly resolvable.

Do not build features around the Mantle endpoint today. Save the bearer token to `~/.aws/bedrock_bearer.env`, revisit when DNS starts returning an address. Standard `bedrock-runtime` SDK is the working path, and it uses AKIA keys not bearer tokens.

## IAM setup gotchas

- **Root user sign-in** (not "IAM user") for account setup. IAM user is for API keys only.
- Attach **`AdministratorAccess`** OR **`AmazonBedrockLimitedAccess`** to the IAM user. Both grant Marketplace auto-subscribe; AdministratorAccess is simpler.
- Account ID (12 digits) is in the top-right account icon after Root sign-in. Not on Settings page.
- **Region is not remembered globally** — Bedrock Console may open `eu-north-1` where Bedrock is not populated. Force `us-east-1` or `us-west-2` in URL: `https://us-east-1.console.aws.amazon.com/bedrock/`.

## Anthropic use-case form

Mandatory on first Anthropic model invoke. Two paths:

1. Console → Bedrock → Model catalog → pick any Claude model → **Submit use case details** button.
2. Programmatic: `bedrock.PutUseCaseForModelAccess(modelId='...', useCase='...')` on the `bedrock` client (NOT `bedrock-runtime`).

The form takes free-text description. After submission, access is granted immediately.

## Quick test script

```bash
export AWS_ACCESS_KEY_ID=<AKIA...>
export AWS_SECRET_ACCESS_KEY=<secret>
export AWS_DEFAULT_REGION=us-east-1

# Polly — should work on day 1
python3 -c "import boto3; p=boto3.client('polly',region_name='us-east-1'); r=p.synthesize_speech(Text='Hello world',OutputFormat='mp3',VoiceId='Joanna'); open('t.mp3','wb').write(r['AudioStream'].read())"

# S3 — day 1
python3 -c "import boto3; s3=boto3.client('s3',region_name='us-east-1'); print('buckets:', len(s3.list_buckets()['Buckets']))"

# STS identity check
python3 -c "import boto3; r=boto3.client('sts',region_name='us-east-1').get_caller_identity(); print(r['Account'], r['Arn'])"

# Bedrock text (Nova Lite) — may fail first hour with Operation not allowed
python3 -c "import boto3; r=boto3.client('bedrock-runtime',region_name='us-east-1').converse(modelId='amazon.nova-lite-v1:0',messages=[{'role':'user','content':[{'text':'hi'}]}],inferenceConfig={'maxTokens':20}); print(r['output'])"

# Nova Canvas image — after Marketplace settles
python3 -c "import boto3,json,base64; r=boto3.client('bedrock-runtime',region_name='us-east-1').invoke_model(modelId='amazon.nova-canvas-v1:0',body=json.dumps({'promptText':'a castle','numberOfImages':1,'imageConfig':{'dimension':'1024x1024'}})); d=json.loads(r['body'].read().decode()); open('i.png','wb').write(base64.b64decode(d['images'][0]['b64Image']))"
```

## When user wants image gen on AWS: pivot away

Bedrock image generation is effectively unavailable on free tier as of 2026:
- Titan Image Generator v1/v2 → EOL
- Nova Canvas → Marketplace-gated, slow settle, unclear pricing
- Stability SDXL → EOL

Offer in this order:
1. **Google Colab + SDXL/Flux** — free T4 GPU, professional quality, unlimited images
2. **Pollinations.ai** — instant, no key, quality modest
3. **HuggingFace Inference** — free signup, ~150 img/day
4. **Polly** for TTS — the one AWS win for media generation. Save AWS credits for VPS/Lambda/S3 workloads.

## Rules / Pitfalls

- **Never** promise Bedrock image gen works out-of-the-box — Titan and SDXL are deprecated.
- **Never** build against `bedrock-mantle.*` endpoints today — DNS NXDOMAIN.
- **Always** check `list_foundation_models()` output to confirm model IDs before writing invocation code.
- **Always** use `region_name='us-east-1'` or `'us-west-2'` — no other regions have full Bedrock coverage.
- **Always** save the bearer token even if Mantle doesn't resolve yet — it's a 365-day credential that becomes useful when the endpoint launches.
- **Always** test with `get_caller_identity()` first to confirm AKIA keys are wired, before debugging model errors.
- **Never** use `aws-marketplace` as a boto3 client name — correct name is `marketplacecommerceanalytics` or `marketplace-agreement`.
- **Never** assume Region persists across Console sessions — check the URL host segment on every landing.
- **Anthropic always needs use case form** on first invoke — do not skip and expect Claude to just work.
