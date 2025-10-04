# Scriberr API Documentation

## Overview

Scriberr exposes a comprehensive REST API for programmatic access to all features. The API supports both JWT authentication and API key authentication.

**Base URL**: `http://localhost:8080/api/v1`

**API Documentation**: 
- Interactive Swagger UI: `/swagger/index.html`
- Online: https://scriberr.app/api.html

## Authentication

### JWT Authentication

JWT tokens are used for user session management and web interface authentication.

#### Login
```bash
# Obtain JWT token
curl -X POST 'http://localhost:8080/api/v1/auth/login' \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "alice",
    "password": "your-password"
  }'

# Response
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_in": 86400
}
```

#### Using JWT Token
```bash
curl -X GET 'http://localhost:8080/api/v1/transcription/list' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIs...'
```

#### Refresh Token
```bash
curl -X POST 'http://localhost:8080/api/v1/auth/refresh' \
  -H 'Content-Type: application/json' \
  -d '{
    "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
  }'
```

### API Key Authentication

API keys are long-lived credentials suitable for automation and integrations.

#### Create API Key
```bash
# Must use JWT token to create API keys
curl -X POST 'http://localhost:8080/api/v1/api-keys/' \
  -H 'Authorization: Bearer YOUR_JWT' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "My Integration",
    "expires_at": "2025-12-31T23:59:59Z"
  }'

# Response
{
  "id": "key_abc123...",
  "key": "sk_live_abc123...",  # Save this! Won't be shown again
  "name": "My Integration",
  "created_at": "2024-01-01T00:00:00Z",
  "expires_at": "2025-12-31T23:59:59Z"
}
```

#### Using API Key
```bash
curl -X GET 'http://localhost:8080/api/v1/transcription/list' \
  -H 'X-API-Key: sk_live_abc123...'
```

#### List API Keys
```bash
curl -X GET 'http://localhost:8080/api/v1/api-keys/' \
  -H 'Authorization: Bearer YOUR_JWT'
```

#### Delete API Key
```bash
curl -X DELETE 'http://localhost:8080/api/v1/api-keys/{id}' \
  -H 'Authorization: Bearer YOUR_JWT'
```

## Core API Endpoints

### Transcription Jobs

#### Upload Audio File

```bash
curl -X POST 'http://localhost:8080/api/v1/transcription/upload' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -F 'file=@recording.mp3' \
  -F 'title=Meeting Recording'

# Response
{
  "id": "job_abc123",
  "title": "Meeting Recording",
  "status": "uploaded",
  "audio_path": "uploads/job_abc123.mp3",
  "created_at": "2024-01-01T10:00:00Z"
}
```

**Supported Formats**: WAV, MP3, FLAC, M4A, OGG, WMA

#### Submit Transcription Job

```bash
curl -X POST 'http://localhost:8080/api/v1/transcription/submit' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "title": "Team Meeting",
    "audio_path": "uploads/audio.mp3",
    "parameters": {
      "model_family": "whisper",
      "model": "medium",
      "device": "cpu",
      "batch_size": 16,
      "language": "en",
      "diarization": true,
      "diarization_model": "pyannote",
      "num_speakers": 3,
      "hf_token": "hf_..."
    }
  }'

# Response
{
  "id": "job_xyz789",
  "title": "Team Meeting",
  "status": "pending",
  "parameters": { ... },
  "created_at": "2024-01-01T10:00:00Z"
}
```

#### Start Transcription

```bash
curl -X POST 'http://localhost:8080/api/v1/transcription/job_abc123/start' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "id": "job_abc123",
  "status": "processing",
  "message": "Transcription started"
}
```

#### Get Job Status

```bash
curl -X GET 'http://localhost:8080/api/v1/transcription/job_abc123/status' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "id": "job_abc123",
  "status": "processing",  # uploaded, pending, processing, completed, failed
  "progress": 45,          # percentage (if available)
  "message": "Transcribing audio..."
}
```

#### Get Transcript

```bash
curl -X GET 'http://localhost:8080/api/v1/transcription/job_abc123/transcript' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "segments": [
    {
      "start": 0.0,
      "end": 2.5,
      "text": "Hello everyone, welcome to the meeting.",
      "speaker": "SPEAKER_00",
      "words": [
        {"word": "Hello", "start": 0.0, "end": 0.3},
        {"word": "everyone", "start": 0.4, "end": 0.8},
        ...
      ]
    },
    ...
  ]
}
```

#### List Jobs

```bash
curl -X GET 'http://localhost:8080/api/v1/transcription/list?page=1&per_page=20&status=completed' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "jobs": [
    {
      "id": "job_abc123",
      "title": "Meeting Recording",
      "status": "completed",
      "created_at": "2024-01-01T10:00:00Z",
      "updated_at": "2024-01-01T10:15:00Z"
    },
    ...
  ],
  "total": 150,
  "page": 1,
  "per_page": 20
}
```

**Query Parameters**:
- `page`: Page number (default: 1)
- `per_page`: Results per page (default: 20, max: 100)
- `status`: Filter by status (uploaded, pending, processing, completed, failed)
- `search`: Search in title

#### Delete Job

```bash
curl -X DELETE 'http://localhost:8080/api/v1/transcription/job_abc123' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "message": "Job deleted successfully"
}
```

#### Kill Running Job

```bash
curl -X POST 'http://localhost:8080/api/v1/transcription/job_abc123/kill' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "message": "Job killed successfully",
  "status": "failed"
}
```

### YouTube Transcription

```bash
curl -X POST 'http://localhost:8080/api/v1/transcription/youtube' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
    "title": "Video Title",
    "parameters": {
      "model": "medium",
      "diarization": true
    }
  }'

# Response
{
  "id": "job_youtube123",
  "title": "Video Title",
  "status": "processing",
  "audio_path": "uploads/job_youtube123.mp3"
}
```

### Quick Transcription

Quick transcription is ephemeral - results are not saved to the database.

```bash
# Submit quick transcription
curl -X POST 'http://localhost:8080/api/v1/transcription/quick' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -F 'file=@audio.mp3' \
  -F 'model=small' \
  -F 'language=en'

# Response
{
  "id": "quick_abc123",
  "status": "processing"
}

# Check status
curl -X GET 'http://localhost:8080/api/v1/transcription/quick/quick_abc123' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response (when complete)
{
  "id": "quick_abc123",
  "status": "completed",
  "transcript": {
    "segments": [ ... ]
  }
}
```

### Profiles

Profiles save transcription configurations for reuse.

#### Create Profile

```bash
curl -X POST 'http://localhost:8080/api/v1/profiles/' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "High Quality English",
    "description": "Best quality for English transcription",
    "parameters": {
      "model_family": "whisper",
      "model": "large-v3",
      "language": "en",
      "batch_size": 8,
      "diarization": true,
      "diarization_model": "pyannote",
      "hf_token": "hf_..."
    },
    "is_default": true
  }'
```

#### List Profiles

```bash
curl -X GET 'http://localhost:8080/api/v1/profiles/' \
  -H 'X-API-Key: YOUR_API_KEY'
```

#### Update Profile

```bash
curl -X PUT 'http://localhost:8080/api/v1/profiles/{id}' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Updated Name",
    "parameters": { ... }
  }'
```

#### Delete Profile

```bash
curl -X DELETE 'http://localhost:8080/api/v1/profiles/{id}' \
  -H 'X-API-Key: YOUR_API_KEY'
```

### Notes

#### Create Note

```bash
curl -X POST 'http://localhost:8080/api/v1/transcription/job_abc123/notes' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "content": "Important decision made here",
    "timestamp": 125.5
  }'

# Response
{
  "id": "note_xyz789",
  "transcription_job_id": "job_abc123",
  "content": "Important decision made here",
  "timestamp": 125.5,
  "created_at": "2024-01-01T10:30:00Z"
}
```

#### List Notes

```bash
curl -X GET 'http://localhost:8080/api/v1/transcription/job_abc123/notes' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "notes": [
    {
      "id": "note_xyz789",
      "content": "Important decision made here",
      "timestamp": 125.5,
      "created_at": "2024-01-01T10:30:00Z"
    },
    ...
  ]
}
```

#### Update Note

```bash
curl -X PUT 'http://localhost:8080/api/v1/notes/note_xyz789' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "content": "Updated note content",
    "timestamp": 126.0
  }'
```

#### Delete Note

```bash
curl -X DELETE 'http://localhost:8080/api/v1/notes/note_xyz789' \
  -H 'X-API-Key: YOUR_API_KEY'
```

### Chat

#### Create Chat Session

```bash
curl -X POST 'http://localhost:8080/api/v1/chat/sessions' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "transcription_id": "job_abc123",
    "title": "Meeting Analysis"
  }'

# Response
{
  "id": "chat_session_123",
  "transcription_id": "job_abc123",
  "title": "Meeting Analysis",
  "created_at": "2024-01-01T11:00:00Z"
}
```

#### Send Chat Message

```bash
curl -X POST 'http://localhost:8080/api/v1/chat/sessions/chat_session_123/messages' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "content": "What were the main action items discussed?",
    "model": "gpt-4",
    "temperature": 0.7
  }'

# Response (Server-Sent Events stream)
data: {"content": "Based"}
data: {"content": " on"}
data: {"content": " the"}
data: {"content": " transcript"}
...
data: {"done": true}
```

#### Get Chat Session

```bash
curl -X GET 'http://localhost:8080/api/v1/chat/sessions/chat_session_123' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "id": "chat_session_123",
  "transcription_id": "job_abc123",
  "title": "Meeting Analysis",
  "messages": [
    {
      "role": "user",
      "content": "What were the main action items?",
      "timestamp": "2024-01-01T11:05:00Z"
    },
    {
      "role": "assistant",
      "content": "Based on the transcript, the main action items are...",
      "timestamp": "2024-01-01T11:05:05Z"
    }
  ]
}
```

#### List Chat Sessions

```bash
curl -X GET 'http://localhost:8080/api/v1/chat/transcriptions/job_abc123/sessions' \
  -H 'X-API-Key: YOUR_API_KEY'
```

#### Delete Chat Session

```bash
curl -X DELETE 'http://localhost:8080/api/v1/chat/sessions/chat_session_123' \
  -H 'X-API-Key: YOUR_API_KEY'
```

### Summarization

#### Summarize Transcript

```bash
curl -X POST 'http://localhost:8080/api/v1/summarize/' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "transcription_id": "job_abc123",
    "template_id": "template_default",
    "model": "gpt-4",
    "temperature": 0.5
  }'

# Response
{
  "summary": "## Overview\n\nThe meeting covered three main topics...",
  "model": "gpt-4",
  "template_name": "Default Summary",
  "created_at": "2024-01-01T12:00:00Z"
}
```

#### Create Summary Template

```bash
curl -X POST 'http://localhost:8080/api/v1/summaries/' \
  -H 'X-API-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Executive Summary",
    "description": "High-level summary for executives",
    "system_prompt": "You are a professional meeting summarizer.",
    "user_prompt": "Create a concise executive summary with:\n- Key decisions\n- Action items\n- Next steps",
    "model": "gpt-4",
    "temperature": 0.3
  }'
```

#### List Summary Templates

```bash
curl -X GET 'http://localhost:8080/api/v1/summaries/' \
  -H 'X-API-Key: YOUR_API_KEY'
```

### LLM Configuration

#### Get LLM Config

```bash
curl -X GET 'http://localhost:8080/api/v1/llm/config' \
  -H 'Authorization: Bearer YOUR_JWT'  # JWT only

# Response
{
  "provider": "openai",  # or "ollama"
  "openai_api_key": "sk-...",
  "openai_base_url": "https://api.openai.com/v1",
  "ollama_base_url": "http://localhost:11434",
  "default_model": "gpt-4"
}
```

#### Save LLM Config

```bash
curl -X POST 'http://localhost:8080/api/v1/llm/config' \
  -H 'Authorization: Bearer YOUR_JWT' \
  -H 'Content-Type: application/json' \
  -d '{
    "provider": "openai",
    "openai_api_key": "sk-...",
    "default_model": "gpt-4"
  }'
```

### Admin

#### Get Queue Statistics

```bash
curl -X GET 'http://localhost:8080/api/v1/admin/queue/stats' \
  -H 'X-API-Key: YOUR_API_KEY'

# Response
{
  "active_workers": 2,
  "queued_jobs": 3,
  "running_jobs": 1,
  "completed_jobs": 150,
  "failed_jobs": 5
}
```

## Transcription Parameters

### Model Families

- **whisper**: OpenAI Whisper models via WhisperX
- **nvidia**: NVIDIA NeMo models (Parakeet, Canary)

### WhisperX Parameters

```json
{
  "model_family": "whisper",
  "model": "medium",                    // tiny, base, small, medium, large-v3
  "device": "cpu",                      // cpu, cuda
  "device_index": 0,                    // GPU index (0-7)
  "batch_size": 16,                     // Batch size for processing
  "compute_type": "float32",            // float32, float16, int8
  "language": "en",                     // Language code or "auto"
  "task": "transcribe",                 // transcribe or translate
  
  // VAD settings
  "vad_method": "pyannote",            // Voice Activity Detection method
  "vad_onset": 0.5,
  "vad_offset": 0.363,
  
  // Alignment
  "align_model": null,                  // Custom alignment model
  "interpolate_method": "nearest",
  "no_align": false,
  
  // Diarization
  "diarization": true,
  "diarization_model": "pyannote",     // pyannote or sortformer
  "min_speakers": null,
  "max_speakers": null,
  "num_speakers": 2,                    // Expected number of speakers
  "hf_token": "hf_...",                // Hugging Face token (required for pyannote)
  
  // Output
  "output_format": "all",              // all, srt, txt, json
  "verbose": true
}
```

### NVIDIA Parakeet Parameters

```json
{
  "model_family": "nvidia",
  "model": "parakeet-ctc-1.1b",
  "device": "cuda",
  "batch_size": 16,
  "language": "en"
}
```

### NVIDIA Canary Parameters

```json
{
  "model_family": "nvidia",
  "model": "canary-1b",
  "device": "cuda",
  "batch_size": 16,
  "language": "en",                    // Supports code-switching
  "task": "asr"                        // asr or translate
}
```

## Error Responses

All errors follow this format:

```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": { }  // Optional additional context
}
```

### Common HTTP Status Codes

- `200 OK`: Success
- `201 Created`: Resource created
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Missing or invalid auth
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource not found
- `409 Conflict`: Resource conflict (e.g., duplicate)
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error

## Rate Limiting

Currently, Scriberr does not enforce API rate limits, but this may change in future versions. Use reasonable request rates to avoid overwhelming the server.

## Pagination

List endpoints support pagination:

```bash
# Query parameters
?page=1          # Page number (1-indexed)
&per_page=20     # Results per page (max 100)
```

Response includes pagination metadata:

```json
{
  "items": [ ... ],
  "total": 150,
  "page": 1,
  "per_page": 20,
  "total_pages": 8
}
```

## Webhooks

Webhooks are not currently supported but are planned for future releases. Subscribe to GitHub issues for updates.

## SDK & Client Libraries

Official SDKs are not yet available. The API follows REST conventions and works with any HTTP client:

**Python Example**:
```python
import requests

api_key = "sk_live_..."
base_url = "http://localhost:8080/api/v1"

# Upload and transcribe
files = {'file': open('audio.mp3', 'rb')}
response = requests.post(
    f"{base_url}/transcription/upload",
    headers={"X-API-Key": api_key},
    files=files
)
job = response.json()

# Start transcription
requests.post(
    f"{base_url}/transcription/{job['id']}/start",
    headers={"X-API-Key": api_key}
)

# Poll for completion
import time
while True:
    status = requests.get(
        f"{base_url}/transcription/{job['id']}/status",
        headers={"X-API-Key": api_key}
    ).json()
    
    if status['status'] in ['completed', 'failed']:
        break
    time.sleep(5)

# Get transcript
transcript = requests.get(
    f"{base_url}/transcription/{job['id']}/transcript",
    headers={"X-API-Key": api_key}
).json()
```

**JavaScript Example**:
```javascript
const apiKey = 'sk_live_...';
const baseUrl = 'http://localhost:8080/api/v1';

// Upload file
const formData = new FormData();
formData.append('file', audioFile);

const uploadResponse = await fetch(`${baseUrl}/transcription/upload`, {
  method: 'POST',
  headers: { 'X-API-Key': apiKey },
  body: formData
});
const job = await uploadResponse.json();

// Start transcription
await fetch(`${baseUrl}/transcription/${job.id}/start`, {
  method: 'POST',
  headers: { 'X-API-Key': apiKey }
});

// Poll for completion
while (true) {
  const statusResponse = await fetch(
    `${baseUrl}/transcription/${job.id}/status`,
    { headers: { 'X-API-Key': apiKey } }
  );
  const status = await statusResponse.json();
  
  if (['completed', 'failed'].includes(status.status)) {
    break;
  }
  await new Promise(r => setTimeout(r, 5000));
}

// Get transcript
const transcriptResponse = await fetch(
  `${baseUrl}/transcription/${job.id}/transcript`,
  { headers: { 'X-API-Key': apiKey } }
);
const transcript = await transcriptResponse.json();
```

## Best Practices

1. **Use API Keys for Automation**: JWT tokens expire, API keys are long-lived
2. **Poll Responsibly**: Use 5-10 second intervals when checking job status
3. **Handle Errors**: Always check HTTP status codes and error responses
4. **Secure Your Keys**: Store API keys securely, never commit to source control
5. **Set Expiration**: Use expiring API keys for better security
6. **Clean Up**: Delete completed jobs you no longer need
7. **Validate Input**: Check file sizes and formats before uploading
8. **Use Profiles**: Create profiles for consistent configurations

## Further Resources

- **Interactive API Docs**: `/swagger/index.html` on your Scriberr instance
- **Source Code**: [github.com/rishikanthc/Scriberr](https://github.com/rishikanthc/Scriberr)
- **Issues & Feature Requests**: GitHub Issues
- **Community Discussions**: GitHub Discussions
