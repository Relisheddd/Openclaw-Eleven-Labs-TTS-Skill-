# ElevenLabs Text-to-Speech Tool

## Overview

You have access to the ElevenLabs API for text-to-speech generation. Use it whenever the user asks you to speak, generate audio, read something aloud, create voiceovers, or produce TTS output.

## Configuration

Your ElevenLabs API key is stored in `/Users/benhersh/.openclaw/openclaw.json` under the key `elevenlabs_api_key` (or similar — parse the JSON and locate it).

Load it like this:

```python
import json

with open("/Users/benhersh/.openclaw/openclaw.json", "r") as f:
    config = json.load(f)

# Try common key names — adapt to your actual JSON structure
api_key = (
    config.get("elevenlabs_api_key")
    or config.get("elevenlabs", {}).get("api_key")
    or config.get("ELEVENLABS_API_KEY")
)
```

## How to Generate Speech

### Step 1: List available voices (if no voice_id is known)

```python
import requests

headers = {"xi-api-key": api_key}
response = requests.get("https://api.elevenlabs.io/v1/voices", headers=headers)

if response.status_code == 200:
    voices = response.json().get("voices", [])
    for v in voices:
        print(f"{v['name']}: {v['voice_id']}")
else:
    print(f"Error {response.status_code}: {response.text}")
```

Use this to discover valid voice IDs. Cache the results so you don't call it repeatedly.

### Step 2: Generate audio

```python
import requests

voice_id = "21m00Tcm4TlvDq8ikWAM"  # Default "Rachel" — replace with user preference

url = f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}"

headers = {
    "xi-api-key": api_key,
    "Content-Type": "application/json",
    "Accept": "audio/mpeg"
}

payload = {
    "text": "Your text goes here.",
    "model_id": "eleven_monolingual_v1",
    "voice_settings": {
        "stability": 0.75,
        "similarity_boost": 0.75,
        "style": 0.0,
        "use_speaker_boost": True
    }
}

response = requests.post(url, json=payload, headers=headers)

if response.status_code == 200:
    output_path = "/Users/benhersh/output_audio.mp3"
    with open(output_path, "wb") as f:
        f.write(response.content)
    print(f"Audio saved to {output_path}")
else:
    print(f"Error {response.status_code}: {response.text}")
```

### Step 3: Play the audio (optional)

```python
import subprocess
# macOS
subprocess.run(["afplay", output_path])
```

## Automatic Troubleshooting

If a TTS call fails, diagnose automatically before reporting to the user:

| Status Code | Meaning | Auto-Fix |
|-------------|---------|----------|
| 401 | Invalid API key | Re-read `/Users/benhersh/.openclaw/openclaw.json`, check key name variants |
| 403 | Key lacks permission | Log error, tell user their plan may not include this voice/model |
| 404 | Invalid voice_id | Call `/v1/voices` to list valid IDs, pick the closest match, retry |
| 422 | Bad payload | Check `text` is non-empty, `voice_settings` values are 0.0–1.0, retry |
| 429 | Rate limited | Read `Retry-After` header, wait that many seconds, retry automatically |
| 500+ | Server error | Wait 3 seconds, retry up to 2 times |

### Troubleshooting script (run this if things break)

```python
import requests, json

# Load key
with open("/Users/benhersh/.openclaw/openclaw.json", "r") as f:
    config = json.load(f)

api_key = (
    config.get("elevenlabs_api_key")
    or config.get("elevenlabs", {}).get("api_key")
    or config.get("ELEVENLABS_API_KEY")
)

print(f"API key loaded: {'Yes' if api_key else 'NO — key not found in config'}")
print(f"Key prefix: {api_key[:8]}..." if api_key else "N/A")

# Test auth
r = requests.get("https://api.elevenlabs.io/v1/user", headers={"xi-api-key": api_key})
print(f"Auth check: {r.status_code}")
if r.status_code == 200:
    user = r.json()
    print(f"Account: {user.get('subscription', {}).get('tier', 'unknown')} tier")
    print(f"Characters remaining: {user.get('subscription', {}).get('character_limit', '?')}")
else:
    print(f"Auth failed: {r.text}")

# List voices
r = requests.get("https://api.elevenlabs.io/v1/voices", headers={"xi-api-key": api_key})
if r.status_code == 200:
    voices = r.json().get("voices", [])
    print(f"\nAvailable voices ({len(voices)}):")
    for v in voices:
        print(f"  {v['name']}: {v['voice_id']}")
else:
    print(f"Voice list failed: {r.text}")
```

## Behavior Rules

1. **Never ask the user for their API key** — it is in the config file. Read it.
2. **Never ask which voice to use unless the user has multiple custom voices** — default to "Rachel" (ID: `21m00Tcm4TlvDq8ikWAM`) or the first available voice.
3. **If a call fails, troubleshoot automatically** using the table above. Only report to the user after you have tried to fix it.
4. **Save audio as .mp3** to a sensible location (e.g., `~/Desktop/` or a project folder if context is clear).
5. **For long text**, split into chunks under 5000 characters, generate each, then concatenate:

```python
from pydub import AudioSegment
import io

chunks = [text[i:i+4500] for i in range(0, len(text), 4500)]
combined = AudioSegment.empty()

for chunk in chunks:
    payload["text"] = chunk
    r = requests.post(url, json=payload, headers=headers)
    if r.status_code == 200:
        combined += AudioSegment.from_mp3(io.BytesIO(r.content))

combined.export("/Users/benhersh/output_audio.mp3", format="mp3")
```

6. **If the user says "say this" or "read this aloud"** — that means generate TTS. Do it immediately.
