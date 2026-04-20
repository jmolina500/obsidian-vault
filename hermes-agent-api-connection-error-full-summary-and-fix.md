## Hermes Agent APIConnectionError — Full Summary & Fix

### Root Cause
The issue was a DNS resolution failure, not Hermes, OpenRouter, or the model. Your system was using systemd-resolved with 127.0.0.53, but Tailscale DNS at 100.100.100.100 took over all DNS queries through "DNS Domain: ~.". This forced all DNS traffic through Tailscale, which was not resolving external domains properly, causing errors like "Could not resolve host: openrouter.ai".

### Symptoms
- Hermes error: APIConnectionError
- Curl error: Could not resolve host
- OpenRouter and Moonshot APIs unreachable
- External API calls failing or timing out

### Fix
Disable Tailscale DNS takeover with:
tailscale up --accept-dns=false

### Verification
Run:
ping openrouter.ai
curl https://openrouter.ai/api/v1/models

Expected result:
- ping resolves the domain
- curl returns JSON

### Optional Cleanup
Run:
resolvectl flush-caches

### Best Practice
For a stable setup with agents, use:
tailscale up --accept-dns=false --accept-routes=true

### Additional Note for Moonshot Models
If using Moonshot kimi models, set temperature to 1. Otherwise you may get:
invalid temperature: only 1 is allowed

### Key Insight
This was not caused by Hermes, OpenRouter, or Moonshot. It was caused by Tailscale DNS override misconfiguration. DNS issues can lead to API failures, retries, timeouts, and model errors.

### Final Outcome
After fixing DNS:
- External APIs became reachable
- Hermes agent worked normally
- Connection errors were resolved
