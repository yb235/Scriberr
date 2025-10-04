# End-to-End Workflows

This guide walks through complete workflows for common use cases in Scriberr.

## Workflow 1: Basic Audio Transcription

### Goal
Upload an audio file and get a transcript with timestamps.

### Steps

#### 1. Start Scriberr

```bash
# Docker
docker run -d -p 8080:8080 -v scriberr_data:/app/data ghcr.io/rishikanthc/scriberr:latest

# Or binary
scriberr
```

Access at http://localhost:8080

#### 2. Register Account

1. Navigate to http://localhost:8080/register
2. Enter username and password
3. Click "Register"
4. You're automatically logged in

#### 3. Upload Audio

1. Click "Upload" or drag-and-drop audio file
2. Supported formats: WAV, MP3, FLAC, M4A, OGG, WMA
3. File uploads to server
4. Job created with status "uploaded"

#### 4. Configure Transcription

**Model Selection**:
- `tiny`: Fastest, lowest quality
- `small`: Good balance
- `medium`: Recommended for most use
- `large-v3`: Best quality, slowest

**Language**:
- Select language (e.g., "en" for English)
- Or leave as "auto" for detection

**Basic Settings**:
- Device: CPU (or CUDA if GPU available)
- Batch Size: 16 (reduce if low on RAM)

#### 5. Start Transcription

1. Click "Start Transcription"
2. Job status changes to "processing"
3. Progress indicator shows completion %

#### 6. Wait for Completion

**Processing time depends on**:
- Audio length
- Model size
- CPU/GPU speed
- System load

**Example times** (10-minute audio):
- tiny (CPU): 2-3 minutes
- medium (CPU): 8-10 minutes
- medium (GPU): 1-2 minutes

#### 7. View Transcript

1. Click on completed job
2. Transcript displays with:
   - Timestamps for each segment
   - Word-level timing
   - Seekable audio player

#### 8. Export Transcript

Choose format:
- **JSON**: Full data with timestamps
- **SRT**: Subtitle format
- **TXT**: Plain text

Click "Download" → Select format

### Complete Example

```bash
# Via API
# 1. Upload
curl -X POST http://localhost:8080/api/v1/transcription/upload \
  -H "X-API-Key: YOUR_KEY" \
  -F "file=@meeting.mp3"

# Response: {"id": "job_abc123", ...}

# 2. Start with parameters
curl -X POST http://localhost:8080/api/v1/transcription/job_abc123/start \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "medium",
    "language": "en",
    "device": "cpu"
  }'

# 3. Check status
curl http://localhost:8080/api/v1/transcription/job_abc123/status \
  -H "X-API-Key: YOUR_KEY"

# 4. Get transcript when complete
curl http://localhost:8080/api/v1/transcription/job_abc123/transcript \
  -H "X-API-Key: YOUR_KEY" > transcript.json
```

## Workflow 2: Multi-Speaker Transcription with Diarization

### Goal
Transcribe a conversation and identify different speakers.

### Prerequisites

**Hugging Face Token**:
1. Create account: https://huggingface.co
2. Accept conditions:
   - https://huggingface.co/pyannote/speaker-diarization-3.0
   - https://huggingface.co/pyannote/segmentation-3.0
3. Create token: https://huggingface.co/settings/tokens
4. Copy token (starts with `hf_`)

### Steps

#### 1. Upload Audio

Same as basic workflow - upload multi-speaker audio file.

#### 2. Configure with Diarization

**Required Settings**:
- ✅ Enable "Diarization"
- Model: Choose Whisper model (e.g., "medium")
- Language: Specify language
- **HF Token**: Paste your Hugging Face token

**Speaker Settings**:
- Number of Speakers: Specify if known (e.g., 2)
- Or set Min/Max range (e.g., 2-4)

**Example Configuration**:
```json
{
  "model": "medium",
  "language": "en",
  "diarization": true,
  "diarization_model": "pyannote",
  "num_speakers": 3,
  "hf_token": "hf_xxxxxxxxxxxxx"
}
```

#### 3. Start & Wait

Processing takes longer with diarization:
- Additional ~2-5 minutes for speaker identification
- Progress shows transcription → diarization phases

#### 4. Review Results

Transcript includes speaker labels:

```json
{
  "segments": [
    {
      "start": 0.0,
      "end": 2.5,
      "text": "Welcome everyone to the meeting.",
      "speaker": "SPEAKER_00"
    },
    {
      "start": 2.6,
      "end": 5.0,
      "text": "Thanks for having me.",
      "speaker": "SPEAKER_01"
    }
  ]
}
```

#### 5. Customize Speaker Labels

1. Click "Edit Speakers"
2. Rename speakers:
   - SPEAKER_00 → "Alice"
   - SPEAKER_01 → "Bob"
   - SPEAKER_02 → "Charlie"
3. Save changes
4. Transcript updates with real names

#### 6. Export with Speaker Names

Download SRT with speaker labels:

```srt
1
00:00:00,000 --> 00:00:02,500
[Alice] Welcome everyone to the meeting.

2
00:00:02,600 --> 00:00:05,000
[Bob] Thanks for having me.
```

## Workflow 3: YouTube Video Transcription

### Goal
Transcribe a YouTube video directly from URL.

### Steps

#### 1. Find YouTube URL

Copy URL from browser, e.g.:
```
https://www.youtube.com/watch?v=dQw4w9WgXcQ
```

#### 2. Paste in Scriberr

1. Click "YouTube" tab
2. Paste URL
3. Optional: Edit title
4. Configure transcription settings

#### 3. Download & Transcribe

Scriberr automatically:
1. Downloads audio using yt-dlp
2. Saves as MP3
3. Starts transcription

#### 4. Wait & View

Same as regular transcription - view results when complete.

### API Example

```bash
curl -X POST http://localhost:8080/api/v1/transcription/youtube \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
    "title": "My Video",
    "parameters": {
      "model": "medium",
      "language": "en"
    }
  }'
```

## Workflow 4: Chat with Transcript

### Goal
Ask questions about a transcript using AI.

### Prerequisites

Configure LLM provider in Settings:

**Option A: OpenAI**
1. Get API key from https://platform.openai.com
2. Settings → LLM → Provider: OpenAI
3. Paste API key
4. Select model (e.g., gpt-4)

**Option B: Ollama (Local)**
1. Install Ollama: https://ollama.ai
2. Pull model: `ollama pull llama2`
3. Settings → LLM → Provider: Ollama
4. Set URL: http://localhost:11434
5. Select model

### Steps

#### 1. Complete Transcription

First, transcribe audio (see Workflow 1 or 2).

#### 2. Open Chat

1. View transcript
2. Click "Chat" button
3. New chat session created

#### 3. Ask Questions

**Example questions**:
- "What were the main topics discussed?"
- "List all action items"
- "Who said [specific quote]?"
- "Summarize the meeting in 3 bullets"

#### 4. Streaming Response

AI responds in real-time:
- Tokens stream as they're generated
- Full conversation history maintained
- Previous messages provide context

#### 5. Multiple Sessions

Create multiple chat sessions:
- Different questions/perspectives
- Compare different LLM models
- Save sessions for later

### API Example

```bash
# 1. Create session
SESSION=$(curl -X POST http://localhost:8080/api/v1/chat/sessions \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "transcription_id": "job_abc123",
    "title": "Meeting Analysis"
  }' | jq -r '.id')

# 2. Send message
curl -X POST http://localhost:8080/api/v1/chat/sessions/$SESSION/messages \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "What were the main action items?",
    "model": "gpt-4",
    "temperature": 0.7
  }'

# Response streams via Server-Sent Events
```

## Workflow 5: Automated Summarization

### Goal
Generate a structured summary using a custom template.

### Steps

#### 1. Create Summary Template

1. Settings → Templates
2. Click "New Template"
3. Configure:

**System Prompt**:
```
You are a professional meeting summarizer. Create concise, 
actionable summaries from meeting transcripts.
```

**User Prompt**:
```
Analyze this transcript and provide:

## Overview
Brief summary of the meeting

## Key Decisions
- List important decisions made

## Action Items
- Who is responsible for what
- Deadlines if mentioned

## Next Steps
What should happen next
```

4. Save template

#### 2. Generate Summary

1. View completed transcript
2. Click "Summarize"
3. Select template
4. Choose model and temperature
5. Click "Generate"

#### 3. Review & Edit

- Summary appears in markdown
- Edit if needed
- Save to transcription record

#### 4. Export

- Download summary as markdown
- Or copy to clipboard

### API Example

```bash
# 1. Create template
TEMPLATE=$(curl -X POST http://localhost:8080/api/v1/summaries/ \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Meeting Summary",
    "system_prompt": "You are a meeting summarizer.",
    "user_prompt": "Create structured summary with key points and action items.",
    "model": "gpt-4",
    "temperature": 0.5
  }' | jq -r '.id')

# 2. Generate summary
curl -X POST http://localhost:8080/api/v1/summarize/ \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "transcription_id": "job_abc123",
    "template_id": "'$TEMPLATE'",
    "model": "gpt-4"
  }'
```

## Workflow 6: Note-Taking During Review

### Goal
Add timestamped notes while reviewing a transcript.

### Steps

#### 1. View Transcript

1. Open completed transcription
2. Audio player + transcript visible

#### 2. Listen & Annotate

As you listen:
1. Pause at important moments
2. Click "Add Note" at current timestamp
3. Type note content
4. Save

**Example notes**:
- "Important decision made here"
- "Follow up with John on this"
- "Great quote for presentation"

#### 3. Navigate via Notes

Click on note → jumps to that timestamp in audio.

#### 4. Edit/Delete Notes

- Hover over note
- Click edit or delete icon
- Update as needed

### API Example

```bash
# Add note at 125.5 seconds
curl -X POST http://localhost:8080/api/v1/transcription/job_abc123/notes \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Important decision made here",
    "timestamp": 125.5
  }'

# List all notes
curl http://localhost:8080/api/v1/transcription/job_abc123/notes \
  -H "X-API-Key: YOUR_KEY"
```

## Workflow 7: Creating Reusable Profiles

### Goal
Save configurations for repeated use.

### Steps

#### 1. Create Profile

1. Settings → Profiles
2. Click "New Profile"
3. Configure all parameters:
   - Model: medium
   - Language: en
   - Diarization: enabled
   - HF Token: (saved securely)
   - Batch size: 16
   - etc.

4. Name: "High Quality English with Speakers"
5. Mark as default (optional)
6. Save

#### 2. Use Profile

When uploading new audio:
1. Select "High Quality English with Speakers" profile
2. All parameters auto-filled
3. Customize if needed
4. Start transcription

#### 3. Multiple Profiles

Create profiles for different scenarios:
- "Quick Preview" (tiny model, no diarization)
- "Production Quality" (large-v3, all features)
- "Spanish Meetings" (medium, es, diarization)
- "Interviews" (large-v2, diarization, verbose)

### API Example

```bash
# Create profile
PROFILE=$(curl -X POST http://localhost:8080/api/v1/profiles/ \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "High Quality English",
    "description": "Best quality for English transcription",
    "is_default": true,
    "parameters": {
      "model": "large-v3",
      "language": "en",
      "diarization": true,
      "num_speakers": 2,
      "hf_token": "hf_xxxxx"
    }
  }' | jq -r '.id')

# Use profile for new job
curl -X POST http://localhost:8080/api/v1/transcription/submit \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "audio_path": "uploads/audio.mp3",
    "profile_id": "'$PROFILE'"
  }'
```

## Workflow 8: Batch Processing Multiple Files

### Goal
Transcribe multiple files efficiently.

### Steps

#### 1. Upload All Files

1. Select multiple files (Ctrl/Cmd + Click)
2. Drag-and-drop to upload area
3. All files upload simultaneously
4. Jobs created with "uploaded" status

#### 2. Apply Same Configuration

**Option A: Use Profile**
1. Create profile with desired settings
2. Select all uploaded jobs
3. Apply profile to all
4. Start all

**Option B: Bulk Configure**
1. Select all jobs
2. Bulk edit settings
3. Apply to all selected
4. Start batch

#### 3. Monitor Progress

- Dashboard shows all jobs
- Filter by status
- See progress for each
- Jobs process in parallel (based on worker count)

#### 4. Review Results

- Jobs complete at different rates
- Review each transcript individually
- Export all as batch

### API Script Example

```bash
#!/bin/bash
# batch-transcribe.sh

API_KEY="YOUR_KEY"
BASE_URL="http://localhost:8080/api/v1"

# Upload all MP3 files in directory
for file in *.mp3; do
  echo "Uploading $file..."
  
  # Upload
  JOB_ID=$(curl -X POST "$BASE_URL/transcription/upload" \
    -H "X-API-Key: $API_KEY" \
    -F "file=@$file" \
    | jq -r '.id')
  
  echo "Job ID: $JOB_ID"
  
  # Start with parameters
  curl -X POST "$BASE_URL/transcription/$JOB_ID/start" \
    -H "X-API-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
      "model": "medium",
      "language": "en"
    }'
  
  echo "Started job $JOB_ID for $file"
done

echo "All files uploaded and started"
```

## Workflow 9: GPU-Accelerated Transcription

### Goal
Use NVIDIA GPU for 5-10x faster transcription.

### Prerequisites

- NVIDIA GPU with CUDA support
- CUDA drivers installed
- Docker with NVIDIA Container Toolkit (for Docker deployment)

### Steps

#### 1. Deploy with GPU Support

**Docker**:
```bash
docker run -d \
  --gpus all \
  --name scriberr \
  -p 8080:8080 \
  -v scriberr_data:/app/data \
  ghcr.io/rishikanthc/scriberr:cuda
```

**Binary**:
```bash
# Ensure CUDA available
nvidia-smi

# Run normally
scriberr
```

#### 2. Configure for GPU

When transcribing:
- Device: **CUDA**
- Device Index: 0 (or GPU number)
- Batch Size: 32 (higher for GPU)
- Compute Type: float16 (faster, minimal quality loss)

**Optional: Use NVIDIA Models**
- Model Family: nvidia
- Model: parakeet-ctc-1.1b (fastest)
- Or: canary-1b (multilingual)

#### 3. Verify GPU Usage

Monitor during transcription:
```bash
watch -n 1 nvidia-smi
```

Should show GPU memory usage and utilization.

#### 4. Benchmark

Test different configurations:

| Config | Time (10 min audio) |
|--------|---------------------|
| medium (CPU) | ~10 minutes |
| medium (GPU, fp32) | ~2 minutes |
| medium (GPU, fp16) | ~1 minute |
| parakeet (GPU) | ~30 seconds |

## Workflow 10: API Integration

### Goal
Integrate Scriberr into your application.

### Steps

#### 1. Generate API Key

1. Login to Scriberr
2. Settings → API Keys
3. Click "Generate Key"
4. Copy key (starts with `sk_live_`)
5. Store securely (won't be shown again)

#### 2. Basic Integration

**Python Example**:

```python
import requests
import time

API_KEY = "sk_live_xxxxx"
BASE_URL = "http://localhost:8080/api/v1"
headers = {"X-API-Key": API_KEY}

# Upload file
with open("audio.mp3", "rb") as f:
    response = requests.post(
        f"{BASE_URL}/transcription/upload",
        headers=headers,
        files={"file": f}
    )
job = response.json()
job_id = job["id"]

# Start transcription
requests.post(
    f"{BASE_URL}/transcription/{job_id}/start",
    headers=headers,
    json={
        "model": "medium",
        "language": "en"
    }
)

# Poll for completion
while True:
    status = requests.get(
        f"{BASE_URL}/transcription/{job_id}/status",
        headers=headers
    ).json()
    
    if status["status"] == "completed":
        break
    elif status["status"] == "failed":
        raise Exception(status.get("error_message"))
    
    time.sleep(5)

# Get transcript
transcript = requests.get(
    f"{BASE_URL}/transcription/{job_id}/transcript",
    headers=headers
).json()

print(transcript)
```

#### 3. Webhook Integration (Future)

Planned for future releases - subscribe to GitHub issues for updates.

## Best Practices

### Performance
1. Use profiles for consistency
2. Start with small models for testing
3. Enable GPU if available
4. Adjust batch size based on RAM

### Organization
1. Use descriptive job titles
2. Create profiles for different use cases
3. Export and backup important transcripts
4. Clean up old jobs regularly

### Quality
1. Specify language (don't rely on auto-detect)
2. Use larger models for important transcripts
3. Review and edit speaker labels
4. Add notes for context

### Security
1. Use API keys for automation
2. Set expiration on keys
3. Rotate keys periodically
4. Don't commit keys to git
5. Use HTTPS in production

## Troubleshooting

### Job Stuck in Processing

1. Check queue status in Admin panel
2. Look at logs for errors
3. Kill and restart job if needed
4. Check system resources (RAM, disk space)

### Poor Transcription Quality

1. Try larger model
2. Specify language explicitly
3. Check audio quality (noise, clarity)
4. Use VAD settings to filter silence

### Diarization Not Working

1. Verify HF token is valid
2. Check accepted model conditions
3. Ensure enough speakers detected
4. Try adjusting min/max speakers

For more help, check the other documentation files or open a GitHub issue!
