# VibeVoice API

VibeVoice exposes a FastAPI service for generating long-form, multi-speaker speech.  It supports
real-time streaming, batch generation, custom voice registration, and SSML‑based prosody control.
This README covers installation, configuration, and client examples for every endpoint in
`api/main.py`.

## Installation
1. Clone the repository and install dependencies:
   ```bash
   git clone https://github.com/microsoft/VibeVoice.git
   cd VibeVoice
   pip install -r requirements.txt
   ```
2. Launch the server (optionally configure concurrency):
   ```bash
   export TTS_MAX_CONCURRENCY=4  # default 1
   uvicorn api.main:app --host 0.0.0.0 --port 8000
   ```

## Request Model
All generation endpoints accept a JSON body with the following schema:

| Field           | Type        | Description |
|----------------|-------------|-------------|
| `script`       | `string`    | Full conversation script. Use `Speaker X:` to denote turns. |
| `speaker_voices`| `string[]` | Voice preset IDs corresponding to each speaker. |
| `cfg_scale`    | `float`     | Classifier-free guidance scale (1.0‑2.0). |
| `is_ssml`      | `bool`      | Set `true` if `script` contains SSML tags. |

## Endpoints

### `GET /`
Health check. Returns service status and whether the TTS model is loaded.

### `POST /api/generate/streaming`
Stream 16‑bit PCM audio chunks over HTTP.
```bash
curl -X POST http://localhost:8000/api/generate/streaming \
  -H "Content-Type: application/json" \
  -d '{
        "script": "Speaker 0: Hello world!",
        "speaker_voices": ["en-Alice_woman"],
        "cfg_scale": 1.3
      }' \
  --output - > output.pcm
```
Python example with `httpx`:
```python
import httpx, soundfile as sf

req = {
    "script": "Speaker 0: Hello world!",
    "speaker_voices": ["en-Alice_woman"],
    "cfg_scale": 1.3,
}
with httpx.stream("POST", "http://localhost:8000/api/generate/streaming", json=req) as r:
    audio = b"".join(r.iter_bytes())
    sf.write("output.wav", audio, 24000, format="RAW", subtype="PCM_16")
```

### `POST /api/generate/batch`
Return a complete WAV file.
```bash
curl -o podcast.wav -X POST http://localhost:8000/api/generate/batch \
  -H "Content-Type: application/json" \
  -d '{
        "script": "Speaker 0: Welcome!",
        "speaker_voices": ["en-Alice_woman"],
        "cfg_scale": 1.3
      }'
```

### `GET ws://<host>/api/generate/ws`
Bidirectional WebSocket endpoint for low‑latency streaming. Send one JSON message after
connection; audio chunks are returned as binary messages.
```python
import asyncio, websockets, soundfile as sf

async def main():
    uri = "ws://localhost:8000/api/generate/ws"
    payload = {
        "script": "Speaker 0: Hello via WebSocket!",
        "speaker_voices": ["en-Alice_woman"],
        "cfg_scale": 1.3,
    }
    async with websockets.connect(uri) as ws:
        await ws.send(json.dumps(payload))
        audio = b""
        async for msg in ws:
            audio += msg
    sf.write("ws_output.wav", audio, 24000, format="RAW", subtype="PCM_16")

asyncio.run(main())
```
Node.js example using `ws`:
```javascript
import WebSocket from "ws";
const ws = new WebSocket("ws://localhost:8000/api/generate/ws");
ws.on("open", () => {
  ws.send(JSON.stringify({
    script: "Speaker 0: Hello from Node!",
    speaker_voices: ["en-Alice_woman"],
    cfg_scale: 1.3,
  }));
});
ws.on("message", data => {
  // append binary chunks to a file or play them
});
```

### `POST /api/voices`
Upload a voice sample and register it for future requests.
```bash
curl -X POST http://localhost:8000/api/voices \
  -F name=MyVoice \
  -F file=@path/to/voice.wav
```
Response: `{ "voice_id": "MyVoice_ab12cd34" }`

## SSML Support
Enable rich prosody control by setting `"is_ssml": true` and embedding SSML in the `script` field.
```json
{
  "script": "<speak>Speaker 0: <emphasis level=\"strong\">Hello</emphasis><break time=\"500ms\"/>world!</speak>",
  "speaker_voices": ["en-Alice_woman"],
  "is_ssml": true
}
```

## Custom Voice Presets
Preload voices by placing audio files under `demo/voices`.  Uploaded voices are stored here and
can be referenced by the returned `voice_id`.

## Concurrency
The server limits concurrent generations via the `TTS_MAX_CONCURRENCY` environment variable. Set a
higher value to exploit multi‑GPU or multi‑core systems.

## Use Cases
- Real‑time assistants that stream speech as text is produced.
- Podcast generation from multi‑speaker scripts.
- On‑the‑fly voice cloning for personalized voices.
- Applications requiring fine‑grained control via SSML.

## License
This API follows the repository's [LICENSE](../LICENSE).
