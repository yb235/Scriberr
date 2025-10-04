# Transcription Engine

## Overview

Scriberr's transcription engine is built on a flexible adapter architecture that supports multiple speech recognition and speaker diarization models. This design allows easy integration of new models while maintaining a consistent API.

## Architecture

### Adapter Pattern

Each transcription or diarization model has a dedicated adapter that implements a common interface:

```go
type TranscriptionAdapter interface {
    GetCapabilities() ModelCapabilities
    GetParameterSchema() []ParameterSchema
    PrepareEnvironment(ctx context.Context) error
    Transcribe(ctx context.Context, params TranscriptionParams) (*TranscriptionResult, error)
    ValidateParameters(params map[string]interface{}) error
}
```

**Benefits**:
- Unified API across different models
- Easy to add new models
- Model-specific optimizations
- Independent environment management

### Model Registry

The registry manages all available adapters:

```go
// Auto-registration in init() functions
func init() {
    registry.GetRegistry().Register(NewWhisperXAdapter())
    registry.GetRegistry().Register(NewPyannoteAdapter())
    registry.GetRegistry().Register(NewParakeetAdapter())
}
```

### Processing Pipeline

1. **Job Submission**: User uploads audio and selects parameters
2. **Queue**: Job added to worker queue
3. **Worker Pickup**: Available worker claims job
4. **Adapter Selection**: Based on `model_family` parameter
5. **Environment Setup**: Prepare Python environment (if needed)
6. **Model Download**: Download model files (if not cached)
7. **Transcription**: Execute model-specific Python script
8. **Result Parsing**: Parse JSON output
9. **Database Update**: Save results and update job status

## Supported Models

### WhisperX (OpenAI Whisper)

**Adapter**: `whisperx_adapter.go`  
**Environment**: `whisperx-env`  
**Model Family**: `whisper`

WhisperX enhances OpenAI's Whisper with:
- Word-level timestamps (alignment)
- Speaker diarization integration
- VAD (Voice Activity Detection)
- Better silence handling

#### Available Models

| Model | Size | Parameters | VRAM | Speed | Quality |
|-------|------|------------|------|-------|---------|
| tiny | 39M | 39M | ~1GB | Fastest | Basic |
| tiny.en | 39M | 39M | ~1GB | Fastest | Basic (English) |
| base | 74M | 74M | ~1GB | Very Fast | Good |
| base.en | 74M | 74M | ~1GB | Very Fast | Good (English) |
| small | 244M | 244M | ~2GB | Fast | Better |
| small.en | 244M | 244M | ~2GB | Fast | Better (English) |
| medium | 769M | 769M | ~5GB | Moderate | Great |
| medium.en | 769M | 769M | ~5GB | Moderate | Great (English) |
| large-v1 | 1550M | 1.55B | ~10GB | Slow | Excellent |
| large-v2 | 1550M | 1.55B | ~10GB | Slow | Excellent |
| large-v3 | 1550M | 1.55B | ~10GB | Slow | Best |

**.en models**: Optimized for English-only transcription

#### Key Features

**Word-Level Timestamps**
- Precise start/end times for each word
- Enables seek-from-text functionality
- Alignment using phoneme recognition

**Language Support**
- 99+ languages supported
- Automatic language detection
- Translation to English (task=translate)

**VAD (Voice Activity Detection)**
- Pyannote-based silence detection
- Reduces hallucinations
- Improves accuracy on noisy audio

**Format Support**
- Input: WAV, MP3, FLAC, M4A, OGG, WMA
- Output: JSON, SRT, TXT, VTT

#### Parameters

```json
{
  "model_family": "whisper",
  "model": "medium",
  "device": "cpu",              // or "cuda"
  "device_index": 0,            // GPU index
  "batch_size": 16,
  "compute_type": "float32",    // float32, float16, int8
  "language": "en",             // or null for auto-detect
  "task": "transcribe",         // or "translate"
  
  // VAD
  "vad_method": "pyannote",
  "vad_onset": 0.5,
  "vad_offset": 0.363,
  
  // Alignment
  "align_model": null,
  "interpolate_method": "nearest",
  "return_char_alignments": false,
  
  // Output
  "output_format": "all",
  "verbose": true
}
```

#### How It Works

1. **Audio Loading**: Load audio file with librosa
2. **VAD**: Detect voice activity, segment audio
3. **Transcription**: Whisper processes each segment
4. **Alignment**: Phoneme-based word alignment
5. **Output**: JSON with word-level timestamps

### Pyannote (Speaker Diarization)

**Adapter**: `pyannote_adapter.go`  
**Environment**: `whisperx-env` (shared)  
**Model Family**: `pyannote`

Pyannote identifies "who spoke when" in audio recordings.

#### Available Models

- **speaker-diarization-3.0**: Latest, most accurate
- **speaker-diarization-3.1**: Improved for noisy audio
- **speaker-diarization**: Legacy model

#### Requirements

**Hugging Face Token**:
1. Create account: https://huggingface.co
2. Accept conditions for:
   - https://huggingface.co/pyannote/speaker-diarization-3.0
   - https://huggingface.co/pyannote/segmentation-3.0
3. Create token: https://huggingface.co/settings/tokens
4. Enable "Repositories" permissions

#### How It Works

1. **Segmentation**: Identify speech regions
2. **Embedding**: Extract speaker embeddings
3. **Clustering**: Group similar voices
4. **Labeling**: Assign speaker labels (SPEAKER_00, SPEAKER_01, etc.)

#### Parameters

```json
{
  "model_family": "pyannote",
  "diarization_model": "pyannote/speaker-diarization-3.0",
  "hf_token": "hf_xxxxx",       // Required!
  "num_speakers": null,          // or specify exact count
  "min_speakers": null,
  "max_speakers": null,
  "device": "cpu"                // or "cuda"
}
```

#### Integration with WhisperX

WhisperX can run diarization automatically:

```json
{
  "model_family": "whisper",
  "model": "medium",
  "diarization": true,
  "diarization_model": "pyannote",
  "num_speakers": 2,
  "hf_token": "hf_xxxxx"
}
```

This produces transcript with speaker labels:

```json
{
  "segments": [
    {
      "start": 0.0,
      "end": 2.5,
      "text": "Hello everyone",
      "speaker": "SPEAKER_00"
    },
    {
      "start": 2.6,
      "end": 5.1,
      "text": "Hi there",
      "speaker": "SPEAKER_01"
    }
  ]
}
```

### NVIDIA Parakeet

**Adapter**: `parakeet_adapter.go`  
**Environment**: `nvidia-env`  
**Model Family**: `nvidia`

Fast ASR model optimized for NVIDIA GPUs.

#### Available Models

- **parakeet-ctc-1.1b**: 1.1B parameters
- **parakeet-ctc-0.6b**: 600M parameters (faster)

#### Features

- GPU-optimized (requires CUDA)
- Very fast inference
- Good accuracy
- English-focused

#### Parameters

```json
{
  "model_family": "nvidia",
  "model": "parakeet-ctc-1.1b",
  "device": "cuda",
  "device_index": 0,
  "batch_size": 16
}
```

#### Performance

- 5-10x faster than Whisper on GPU
- Lower VRAM requirements
- Trade-off: slightly lower accuracy

### NVIDIA Canary

**Adapter**: `canary_adapter.go`  
**Environment**: `nvidia-env`  
**Model Family**: `nvidia`

Multilingual model with code-switching support.

#### Features

- **Multilingual**: English, Spanish, German, French
- **Code-Switching**: Handle mixed languages
- **Translation**: ASR with translation
- **Fast**: GPU-optimized

#### Parameters

```json
{
  "model_family": "nvidia",
  "model": "canary-1b",
  "device": "cuda",
  "language": "en",
  "task": "asr"              // or "translate"
}
```

### NVIDIA Sortformer (Diarization)

**Adapter**: `sortformer_adapter.go`  
**Environment**: `nvidia-env`  
**Model Family**: `nvidia`

GPU-accelerated speaker diarization.

#### Features

- Streaming diarization
- Real-time capable
- GPU-accelerated
- Multi-speaker support

#### Available Models

- **diar_streaming_sortformer_4spk-v2**: Up to 4 speakers

#### Parameters

```json
{
  "model_family": "nvidia",
  "diarization_model": "sortformer",
  "device": "cuda",
  "num_speakers": 4
}
```

## Python Environment Management

### UV Package Manager

Scriberr uses `uv` (ultra-fast Python package installer) for environment management:

- **Fast**: 10-100x faster than pip
- **Reliable**: Lock files for reproducibility
- **Isolated**: Separate environments per model family

### Environment Structure

**WhisperX Environment**:
```
whisperx-env/
├── pyproject.toml       # Dependencies
├── uv.lock             # Locked versions
└── .venv/              # Virtual environment
    └── lib/python3.11/
```

**NVIDIA Environment**:
```
nvidia-env/
├── pyproject.toml      # NeMo dependencies
├── uv.lock
└── .venv/
```

### pyproject.toml Example

```toml
[project]
name = "whisperx-transcription"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "whisperx @ git+https://github.com/m-bain/whisperx.git",
    "torch",
    "torchaudio",
    "transformers",
    "faster-whisper",
    "pyannote.audio"
]
```

### Environment Lifecycle

1. **Check**: Does environment exist?
2. **Create**: `uv venv` creates virtual environment
3. **Install**: `uv pip install` installs dependencies
4. **Cache**: Reuse environment across jobs
5. **Update**: Re-create if dependencies change

## Model Download & Caching

### Whisper Models

**Location**: `~/.cache/huggingface/hub/`

Models auto-download on first use:
- tiny: ~39MB
- base: ~74MB
- small: ~244MB
- medium: ~769MB
- large: ~1.5GB

### Pyannote Models

**Location**: `~/.cache/torch/pyannote/`

Requires Hugging Face authentication:
- speaker-diarization-3.0: ~100MB
- segmentation-3.0: ~80MB

### NVIDIA Models

**Location**: `nvidia-env/` or `~/.cache/torch/NeMo/`

Downloaded via NeMo framework:
- parakeet-ctc-1.1b: ~1.1GB
- canary-1b: ~1GB
- sortformer: ~200MB

### Cache Management

Models are cached permanently unless manually deleted:

```bash
# Clear Whisper cache
rm -rf ~/.cache/huggingface/hub/

# Clear Pyannote cache
rm -rf ~/.cache/torch/pyannote/

# Clear NVIDIA cache
rm -rf nvidia-env/models/
```

## Transcription Workflow

### Step-by-Step Process

#### 1. Audio Upload
```
User uploads audio.mp3
    ↓
Server saves to uploads/job_abc123.mp3
    ↓
Database record created with status="uploaded"
```

#### 2. Job Configuration
```
User selects:
  - Model: medium
  - Language: en
  - Diarization: true
    ↓
Parameters validated against schema
    ↓
Job updated with parameters
```

#### 3. Job Start
```
POST /api/v1/transcription/job_abc123/start
    ↓
Status changed to "pending"
    ↓
Job added to queue
```

#### 4. Worker Pickup
```
Worker claims job from queue
    ↓
Status changed to "processing"
    ↓
Create execution record
```

#### 5. Environment Preparation
```
Check if whisperx-env exists
    ↓
If not: Create environment
    ↓
Install dependencies with uv
    ↓
Download models (if needed)
```

#### 6. Transcription Execution
```
Generate Python script:
    - Load audio
    - Initialize Whisper
    - Run transcription
    - Run diarization (if enabled)
    - Output JSON
    ↓
Execute: uv run python script.py
    ↓
Capture stdout (JSON result)
    ↓
Capture stderr (logs/errors)
```

#### 7. Result Processing
```
Parse JSON output
    ↓
Extract segments, words, speakers
    ↓
Save to database
    ↓
Update job status="completed"
    ↓
Update execution record
```

#### 8. User Retrieval
```
GET /api/v1/transcription/job_abc123/transcript
    ↓
Return formatted transcript
    ↓
Frontend displays with audio player
```

## Output Format

### JSON Structure

```json
{
  "segments": [
    {
      "start": 0.0,
      "end": 2.5,
      "text": "Hello everyone, welcome to the meeting.",
      "speaker": "SPEAKER_00",
      "words": [
        {
          "word": "Hello",
          "start": 0.0,
          "end": 0.3,
          "score": 0.95
        },
        {
          "word": "everyone",
          "start": 0.4,
          "end": 0.8,
          "score": 0.93
        }
        // ... more words
      ]
    }
    // ... more segments
  ],
  "language": "en",
  "language_probability": 0.99
}
```

### SRT Format

```srt
1
00:00:00,000 --> 00:00:02,500
[SPEAKER_00] Hello everyone, welcome to the meeting.

2
00:00:02,600 --> 00:00:05,100
[SPEAKER_01] Hi there, thanks for having me.
```

### TXT Format

```
[00:00:00] SPEAKER_00: Hello everyone, welcome to the meeting.
[00:00:02] SPEAKER_01: Hi there, thanks for having me.
```

## Performance Optimization

### CPU Optimization

**Batch Size**: Larger batches = faster, more memory
```json
{
  "batch_size": 16    // Good for 8GB RAM
}
```

**Compute Type**: Lower precision = faster, less accurate
```json
{
  "compute_type": "int8"    // Fastest, lowest quality
}
```

**Threading**: Utilize multiple cores
```json
{
  "threads": 4    // Match your CPU cores
}
```

### GPU Optimization

**Enable CUDA**:
```json
{
  "device": "cuda",
  "device_index": 0
}
```

**FP16 Precision**: 2x faster, minimal quality loss
```json
{
  "compute_type": "float16"
}
```

**Larger Batches**: GPUs handle larger batches efficiently
```json
{
  "batch_size": 32
}
```

### Model Selection

**Speed vs Quality Trade-off**:

| Use Case | Model | Reason |
|----------|-------|--------|
| Quick preview | tiny | 10x faster |
| Real-time needs | base | Good balance |
| General use | small/medium | Best balance |
| High accuracy | large-v3 | Best quality |
| GPU available | parakeet | 5x faster than Whisper |

## Error Handling

### Common Errors

**Out of Memory**:
```
Error: CUDA out of memory
Solution: Reduce batch_size or use smaller model
```

**Model Download Failed**:
```
Error: Failed to download model
Solution: Check internet connection, verify Hugging Face token
```

**Invalid Audio Format**:
```
Error: Could not load audio file
Solution: Convert to supported format (WAV, MP3, FLAC)
```

**Diarization Failed**:
```
Error: Pyannote token invalid
Solution: Verify HF token, accept model conditions
```

### Debugging

**Enable Verbose Logging**:
```json
{
  "verbose": true
}
```

**Check Job Execution**:
```bash
GET /api/v1/transcription/:id/execution
```

Returns:
```json
{
  "started_at": "2024-01-01T10:00:00Z",
  "completed_at": "2024-01-01T10:15:00Z",
  "processing_duration_seconds": 900,
  "status": "completed",
  "error_message": null,
  "logs": "..."
}
```

## Multi-Track Support

### Audacity Projects

Scriberr can process multi-track Audacity projects (.aup files):

1. Upload .aup file
2. Scriberr extracts individual tracks
3. Each track transcribed separately
4. Transcripts merged into single timeline
5. Speaker labels preserved

### Workflow

```
Upload project.aup
    ↓
Extract tracks: track_0.wav, track_1.wav
    ↓
Transcribe each track
    ↓
Merge transcripts by timestamp
    ↓
Display unified timeline
```

## Future Enhancements

- **Real-time transcription**: Streaming audio support
- **Custom models**: Fine-tuned model integration
- **More languages**: Expand language support
- **GPU pooling**: Shared GPU resources
- **Batch diarization**: Process multiple files together
- **Custom VAD**: Configurable voice detection
- **Post-processing**: Auto-correct, formatting rules
- **Export formats**: More output formats (DOCX, PDF)

## Best Practices

1. **Start Small**: Use tiny/base for testing, medium for production
2. **Enable GPU**: 5-10x speedup if available
3. **Specify Language**: Faster than auto-detection
4. **Use Diarization**: Essential for multi-speaker audio
5. **Cache Models**: First run downloads, subsequent runs are fast
6. **Monitor Resources**: Watch CPU/GPU/RAM usage
7. **Test Parameters**: Find optimal settings for your audio
8. **Clean Up**: Delete old jobs to free disk space
