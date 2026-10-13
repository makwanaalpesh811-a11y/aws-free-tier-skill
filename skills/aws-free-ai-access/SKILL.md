---
name: aws-free-ai-access
description: "Free VPS + Free AI API (2026)"
version: 1.0.0
author: Alpesh Makwana
license: MIT
platforms: [linux]
---

name: aws-free-ai-access
version: 1.0
description: "Free VPS + free AI API tiers for 24/7 Hermes agent."
tags: [aws, vps, free-tier, llm-api, hosting, setup]
related_skills: [hermes-agent, telegram-gateway-setup]
author: Alpesh Makwana
license: MIT
platforms: [linux, macos]
---

# Free VPS + Free AI API (2026)

Goal: user Termux pe hai, but 24/7 agent + AI model chahiye bina kharche ke. Ye skill best free options + commands + billing traps cover karta hai.

## TL;DR Recommendation

| Need | Best free option | Why |
|------|------------------|-----|
| Always-on VPS (24/7) | **Oracle Cloud** Always Free | Only truly perpetual. 2 OCPU / 12GB RAM A1 always-on. |
| Free tier with credits | **AWS Free Plan** | $200 credits for 6 months. Burst projects. |
| Small always-on (US regions only) | **GCP e2-micro** | 1GB RAM, free forever, 3 US regions only. |
| Free AI model API (fast) | **Groq** | 14,400 RPD, no card, OpenAI-compatible. |
| Free AI model API (biggest daily) | **Cerebras** | 1M tokens/day, ultra-fast. |
| Free AI model API (stable catalog) | **Mistral** Experiment | ~1B tokens/month, devstral coding. |
| Free AI (no card, one key) | **OpenRouter** free | 300+ models, filter `:free`. |

## AWS Free Tier (updated 2026)

**CRITICAL UPDATE (15 July 2025+):** New AWS accounts get **$200 credits** ($100 signup + up to $100 via onboarding), NOT 750 hrs/month t2.micro forever. Credits expire 12 months. **Legacy accounts (pre 15 Jul 2025) keep 750 hrs/month t3.micro for 12 months.**

### AWS CLI setup
```bash
pip install awscli
aws configure
# Enter: access key, secret key, region (ap-south-1 for India)
```

### Launch free EC2 instance
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/free_vps_key

aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --count 1 \
  --instance-type t3.micro \
  --key-name free_vps_key \
  --security-group-ids sg-xxxx \
  --region ap-south-1

aws ec2 describe-instances --region ap-south-1 \
  --query 'Reservations[0].Instances[0].PublicIpAddress'

ssh -i ~/.ssh/free_vps_key.pem ec2-user@<PUBLIC_IP>
```

### Billing traps
1. **Only ONE micro instance 24/7** = 750 hrs free. Two = overage charges.
2. **Region** — ap-south-1 (Mumbai) for India, us-east-1 cheapest rates.
3. **EBS storage** — 30GB free, extra billed.
4. **Data egress** — 1GB/month free, then $0.09/GB. Cloudflare tunnel avoids.
5. **Set $10 billing alert immediately**: Console → Billing → Budgets.
6. **12 months later** = full pay-as-you-go NO warning. Calendar reminder.
7. **Free Plan auto-closes** 6 months if unused.

## Oracle Cloud (BEST for 24/7 always-free)

**Truly perpetual, no expiry, no card needed after verification.**

Specs (Always Free):
- 2 Ampere A1 (Arm) OCPUs + 12GB RAM total
- OR 2 AMD micro VMs (1/8 OCPU, 1GB RAM each)
- 200GB block storage, 5 volume backups
- **10TB outbound data transfer/month** (vs AWS 1GB)

### Setup
1. Sign up: https://cloud.oracle.com/free
2. **Pick home region DURING signup** — cannot change later! Pick Singapore/Jakarta for Asia.
3. Verify email + phone
4. Console → Compute → Instances → Create Instance
5. Select "Always Free" shape (A1.1.1 or equivalent)
6. Add SSH key, configure subnet, launch

Pros: No expiry, 10TB egress, no card.
Cons: Waitlist in popular regions, region locked at signup, occasional maintenance.

## GCP Free Tier

- **e2-micro**: 0.25 vCPU burstable, 1GB RAM, 30GB disk, 1GB egress/month
- **Always free, no expiry**
- **Region-restricted**: us-west1, us-central1, us-east1 ONLY
- 1GB RAM tight for Docker

## Free AI Model APIs (2026 verified)

### Groq (RECOMMENDED — fastest, no card)
```bash
# Get key: https://console.groq.com/keys
echo "export GROQ_API_KEY=gsk_xxx" >> ~/.bashrc

python3 -c "
from openai import OpenAI
client = OpenAI(base_url='https://api.groq.com/openai/v1', api_key='GROQ_API_KEY')
resp = client.chat.completions.create(
    model='llama-3.3-70b-versatile',
    messages=[{'role':'user','content':'Hello'}]
)
print(resp.choices[0].message.content)
"
```

Free tier: 30 RPM, 14,400 RPD, 6,000 TPM (Llama 8B). 12 models incl. llama-3.3-70b, qwen3-32b, gpt-oss-120b, whisper.

### Cerebras (biggest daily free)
- 1M tokens/day free across active models
- Key: https://cerebras.ai
- Base URL: `https://api.cerebras.ai/v1`
- Models: Llama 3.1 70B, gpt-oss-120b, glm-4.7
- **Caution**: catalog volatile — models dropped without notice

### Mistral Experiment Plan
- ~1B tokens/month free, no card
- Key: https://console.mistral.ai
- Base URL: `https://api.mistral.ai/v1`
- Best for: coding (devstral), reasoning (magistral)
- Largest stable free catalog

### OpenRouter (one key, 300+ models)
- Key: https://openrouter.ai/keys
- Base URL: `https://openrouter.ai/api/v1`
- Free models end `:free`
- Filter: `https://openrouter.ai/models?max_price=0`
- Anthropic models need $10 topup

### Gemini (free but per-model gotcha)
- Free tier but quota differs per model. flash-lite/pro may return quota-0.
- Use `gemini-flash` for reliable free access.

## Hermes Integration

```yaml
# Hermes config.yaml, providers.custom:
groq:
  base_url: https://api.groq.com/openai/v1
  api_key: ${GROQ_API_KEY}
  model: llama-3.3-70b-versatile

cerebras:
  base_url: https://api.cerebras.ai/v1
  api_key: ${CEREBRAS_API_KEY}
  model: llama3.1-70b

mistral:
  base_url: https://api.mistral.ai/v1
  api_key: ${MISTRAL_API_KEY}
  model: devstral-small
```

## Recommended stack for Hermes trading agent

1. **VPS**: Oracle Cloud A1 (2 OCPU / 12GB) — run Hermes 24/7
2. **AI model**: Groq (Llama 3.3 70B) primary, Cerebras fallback
3. **Tunnel**: Cloudflare Tunnel (free) instead of public SSH/nginx
4. **Storage**: Oracle 200GB free for backtest data

## Rules / Pitfalls

- **Never** auto-upgrade billing — set budget alert instead.
- **Never** >1 AWS micro instance simultaneously.
- **Always** Oracle for 24/7 (AWS credits expire).
- **Always** Cloudflare Tunnel, not public SSH/HTTPS.
- **Never** single free AI API — 2 providers, key rotation.
- **Never** keys in scripts — env vars or Hermes dot-env.
- Oracle region PERMANENT — pick Singapore/Jakarta for India.
- Free AI: check TPM not just RPD — TPM is real ceiling.
- Groq deprecates models silently in 2026 — pin model names not "latest".

## Verification checklist

After setup:
- [ ] AWS: $10 billing alert set up
- [ ] AWS: only 1 instance running (describe-instances)
- [ ] Oracle: A1 shape, not VM.Standard2.A1 (paid)
- [ ] Groq: test call returns content, not 401/429
- [ ] Cerebras: model list confirmed current month
- [ ] Hermes: custom provider responds to ping
- [ ] All API keys in env vars, not hardcoded
