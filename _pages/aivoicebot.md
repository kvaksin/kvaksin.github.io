---
title: AI Voice Bot
date: 2025-07-13
layout: post
---

**Everything is described below is based on Open Source**

# ☎️ Whisper Voice Bot with SIP + CRM Integration

Build a real-time AI voice bot using Open Source Whisper that:
- Listens to live SIP audio (via Jambonz)
- Transcribes user speech using Whisper
- Detects intent and responds with TTS
- Pushes conversation data to CRM
- Escalates call to human agent with context

---

## 🛠️ Components

| Component     | Tool/Tech                           |
|---------------|-------------------------------------|
| SIP Gateway   | Jambonz (or Twilio, Asterisk)       |
| ASR           | Whisper (faster-whisper)            |
| TTS           | Coqui TTS or Bark                   |
| NLU           | Simple Python rules or Rasa         |
| CRM           | HubSpot, Salesforce, Zendesk, etc.  |
| Integration   | Flask + WebSocket + REST            |

---

## 📞 Call Flow

```text
Inbound Call
   ↓
Jambonz SIP app
   ↓
POST /voice → respond with stream + greeting
   ↓
WebSocket audio stream (μ-law PCM)
   ↓
Whisper STT → Bot logic → TTS
   ↓
If escalation needed:
- Send summary to CRM
- Transfer call to agent
```

---

## 🧪 Flask Endpoint: /voice

```python
@app.route("/voice", methods=["POST"])
def voice():
    return jsonify([
        { "verb": "say", "text": "Hi, this is the voice assistant. How can I help you?" },
        {
            "verb": "stream",
            "url": "ws://yourserver.com/audio",
            "mixType": "mono",
            "tracks": ["in"]
        }
    ])
```

---

## 🌐 WebSocket: Receive Audio + Transcribe

```python
@websockets.serve
async def handle_call(websocket):
    audio_buffer = bytearray()

    async for message in websocket:
        msg = json.loads(message)

        if msg.get("event") == "media":
            audio_data = base64.b64decode(msg["media"]["payload"])
            pcm = np.frombuffer(audio_data, dtype=np.uint8).astype(np.int16) - 128
            audio_buffer.extend(pcm.tobytes())

        elif msg.get("event") == "stop":
            sf.write("call.wav", np.frombuffer(audio_buffer, dtype=np.int16), 8000)
            segments, _ = whisper_model.transcribe("call.wav")
            text = " ".join([seg.text for seg in segments])
            print("User said:", text)

            intent = detect_intent(text)
            summary = summarize_session(text, intent)

            send_to_crm({
                "session_id": call_id,
                "summary": summary,
                "intent": intent,
                "transcript": text
            })

            if intent == "transfer":
                transfer_call_to_agent()
            else:
                respond_with_tts("How else can I help you?")
```

---

## 💬 Intent Detection

```python
def detect_intent(text):
    text = text.lower()
    if "order" in text:
        return "order_status"
    elif "speak to agent" in text or "human":
        return "transfer"
    else:
        return "unknown"
```

---

## 🧠 Send Data to CRM

```python
def send_to_crm(data):
    payload = {
        "properties": {
            "hs_note_body": f"[Voicebot Summary]\n{data['summary']}",
            "hs_timestamp": int(time.time() * 1000)
        }
    }
    headers = { "Authorization": f"Bearer {HUBSPOT_TOKEN}" }
    requests.post(HUBSPOT_NOTE_API, headers=headers, json=payload)
```

---

## 📡 Escalate Call to Agent

In `/voice`, redirect to escalation:

```json
{ "verb": "redirect", "url": "https://yourdomain.com/escalate" }
```

In `/escalate`, return:

```json
[
  {
    "verb": "dial",
    "targets": [{ "type": "user", "name": "agent-alex" }]
  }
]
```

---

## 🧠 What Gets Sent to the Agent?

| Field       | Example                     |
|-------------|-----------------------------|
| Phone       | `+15551234567`              |
| Intent      | `order_status`              |
| Summary     | `"Asked about ORD1234"`     |
| Transcript  | `"Where is my order 1234?"` |
| Confidence  | `0.91`                      |

This data is posted to CRM and optionally shown in a **screen pop** when the agent answers the call.

---

## 🎯 Deployment Tips

- Use `ngrok` to test webhooks with Jambonz
- Use GPU for `faster-whisper` real-time transcription
- Run TTS in async mode to avoid latency
- Use Redis/DB to store session context
- Ensure CRM API tokens are securely stored

---

## 📦 Optional Features

- [ ] Record full call audio to CRM
- [ ] Reuse Whisper segments to show speaker turns
- [ ] Add voice activity detection for open-mic
- [ ] Use GPT-based NLU for better intent extraction
- [ ] CRM webhooks to trigger post-call workflows
- [ ] Inteligent call routing to handle different Customer intent (with analysing Customers communication. More details are [here](/pages/aicc))

---

## 🧰 Useful Libraries

```bash
pip install faster-whisper websockets flask soundfile numpy webrtcvad TTS
```

---

## 🔗 References

- [Jambonz Docs](https://www.jambonz.org/docs/)
- [OpenAI Whisper](https://github.com/openai/whisper)
- [Coqui TTS](https://github.com/coqui-ai/TTS)
- [HubSpot CRM API](https://developers.hubspot.com/docs/api/crm/notes)

---

## ✅ Summary

You now have a working architecture for:
- Streaming SIP call audio to Whisper
- Generating real-time bot responses
- Escalating to live agents
- Logging everything to your CRM

Build it modularly, test early, and scale out from here.

date: 2025-07-13