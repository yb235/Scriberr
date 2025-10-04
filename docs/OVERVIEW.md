# Scriberr - Comprehensive Overview

## What is Scriberr?

Scriberr is a self-hosted, offline audio transcription application that converts audio recordings into text with high accuracy. It provides a complete solution for transcribing audio files, identifying speakers, taking notes, and interacting with transcripts using AI.

### Key Highlights

- **Privacy-First**: All processing happens on your machine - no data sent to the cloud
- **Self-Hosted**: Run on your own infrastructure with full control
- **Open Source**: Built with transparency and community collaboration
- **Modern Stack**: React frontend + Go backend = fast, efficient, single binary
- **AI-Powered**: Uses state-of-the-art WhisperX and NVIDIA models for transcription
- **LLM Integration**: Chat with your transcripts using OpenAI or Ollama (local)

## Core Features

### 1. Audio Transcription
- **Multiple Model Support**: WhisperX (OpenAI Whisper), NVIDIA Parakeet, Canary
- **Accuracy**: Word-level timestamps with high precision
- **Flexible**: Works on CPU (no GPU required) or GPU-accelerated
- **Format Support**: WAV, MP3, FLAC, M4A, OGG, WMA

### 2. Speaker Diarization
- **Speaker Identification**: Automatically detect and label different speakers
- **Technologies**: Pyannote, NVIDIA Sortformer
- **Manual Override**: Edit speaker labels after transcription
- **Multi-Speaker**: Handle conversations with multiple participants

### 3. Transcript Features
- **Interactive Reader**: Clean, distraction-free interface
- **Playback Follow-Along**: Transcript highlights as audio plays
- **Seek from Text**: Click any word to jump to that point in audio
- **Multiple Formats**: Export as JSON, SRT, TXT, and more

### 4. Note-Taking & Highlights
- **Timestamp Notes**: Add notes linked to specific moments in audio
- **Quick Navigation**: Jump from note directly to audio position
- **Highlights**: Mark important sections for easy reference

### 5. AI-Powered Analysis
- **Summarization**: Generate summaries using custom templates
- **Chat Interface**: Ask questions about your transcripts
- **LLM Flexibility**: Use OpenAI API or local Ollama models
- **Custom Prompts**: Create reusable summary templates

### 6. YouTube Integration
- **Direct Transcription**: Paste a YouTube URL and transcribe
- **Audio Extraction**: Uses yt-dlp to download audio
- **Same Features**: Full transcript, diarization, chat, notes

### 7. Multi-Track Support
- **Audacity Projects**: Upload .aup files with multiple tracks
- **Individual Processing**: Transcribe each track separately
- **Merge & Combine**: Combine transcripts into single timeline

### 8. REST API
- **Complete Coverage**: All features available via API
- **Authentication**: JWT tokens or API keys
- **OpenAPI/Swagger**: Full API documentation
- **Integration Ready**: Easy to integrate with other tools

## Technology Stack

### Backend (Go)
- **Framework**: Gin (HTTP router)
- **Database**: SQLite with GORM ORM
- **Authentication**: JWT tokens + API keys
- **Queue System**: Custom job queue with auto-scaling workers
- **Process Management**: Python environment orchestration

### Frontend (React)
- **UI Framework**: React 18 with TypeScript
- **Styling**: Tailwind CSS + shadcn/ui components
- **State Management**: React Context API
- **Build Tool**: Vite
- **Audio Player**: Custom HTML5 audio implementation

### Transcription Engine (Python)
- **WhisperX**: OpenAI Whisper with enhanced alignment
- **NVIDIA NeMo**: Parakeet and Canary models for GPU acceleration
- **Pyannote**: Speaker diarization
- **Package Manager**: UV (fast Python package installer)
- **Environment**: Isolated Python environments per model family

### LLM Integration
- **OpenAI**: GPT models via official API
- **Ollama**: Local LLM support for privacy
- **Streaming**: Real-time response streaming
- **Context**: Full transcript included in prompts

## Architecture Patterns

### Single Binary Distribution
Scriberr compiles the React frontend into static files that are embedded directly into the Go binary. This means:
- One executable contains everything
- No separate web server needed
- Easy deployment and distribution
- Simplified updates

### Adapter Pattern for Models
Each transcription/diarization model has an adapter that implements a common interface:
- Easy to add new models
- Consistent API across different engines
- Runtime model switching
- Independent Python environments

### Job Queue System
Transcription jobs are processed asynchronously:
- Auto-scaling worker pool based on CPU
- Graceful job cancellation
- Process tracking and termination
- Priority queue support

### Database-First Design
All data persists in SQLite:
- Transcription jobs and results
- User accounts and authentication
- API keys and sessions
- Notes, summaries, chat history
- System configuration

## Target Users

### Researchers & Students
- Transcribe interviews and lectures
- Generate study summaries
- Search through audio archives
- Privacy-sensitive academic work

### Content Creators
- Transcribe podcasts and videos
- Create show notes automatically
- Generate content summaries
- Improve accessibility with captions

### Businesses
- Meeting transcription
- Customer interview analysis
- Training material transcription
- Compliance and record-keeping

### Privacy-Conscious Users
- No cloud dependency
- Data stays on your infrastructure
- Open source - verify the code
- Self-hosted - full control

## Deployment Options

### 1. Homebrew (macOS/Linux)
```bash
brew tap rishikanthc/scriberr
brew install scriberr
scriberr
```

### 2. Docker
```bash
docker run -d \
  --name scriberr \
  -p 8080:8080 \
  -v scriberr_data:/app/data \
  ghcr.io/rishikanthc/scriberr:latest
```

### 3. Docker Compose
Pre-configured setup with volume management

### 4. Binary Download
Direct download from GitHub releases

### 5. Build from Source
Full control over build process

## System Requirements

### Minimum
- CPU: 2+ cores
- RAM: 4GB
- Storage: 10GB
- OS: Linux, macOS, Windows
- Python: 3.11+ (managed automatically)

### Recommended
- CPU: 4+ cores
- RAM: 8GB
- Storage: 20GB+
- GPU: NVIDIA GPU with CUDA support (optional, for acceleration)

## Getting Started

1. **Install**: Choose your preferred installation method
2. **Start Server**: Run `scriberr` or start Docker container
3. **Access UI**: Open http://localhost:8080
4. **Register**: Create your first user account
5. **Upload Audio**: Drop an audio file or paste YouTube URL
6. **Configure**: Choose model, enable diarization if needed
7. **Transcribe**: Start processing and wait for results
8. **Explore**: Read transcript, add notes, chat with AI

## Security & Privacy

- **Local Processing**: All transcription happens on your machine
- **Encrypted Storage**: Passwords hashed with bcrypt
- **JWT Authentication**: Secure session management
- **API Keys**: Scoped access for integrations
- **No Telemetry**: No tracking or analytics
- **Data Ownership**: You own all data

## Performance

- **CPU Mode**: Works on any modern CPU, slower but accessible
- **GPU Mode**: 5-10x faster with NVIDIA GPU
- **Batch Processing**: Queue multiple files
- **Auto-scaling**: Worker pool adapts to system resources
- **Caching**: Downloaded models cached for reuse

## License

MIT License - Free for personal and commercial use

## Community & Support

- **GitHub Issues**: Bug reports and feature requests
- **Discussions**: Questions and community support
- **Documentation**: https://scriberr.app/docs
- **API Reference**: https://scriberr.app/api.html
- **Changelog**: https://scriberr.app/changelog.html

## What's Next?

Explore the detailed documentation:
- [ARCHITECTURE.md](ARCHITECTURE.md) - Deep dive into system design
- [API.md](API.md) - Complete API reference
- [TRANSCRIPTION.md](TRANSCRIPTION.md) - How transcription works
- [DEVELOPMENT.md](DEVELOPMENT.md) - Contributing guide
- [WORKFLOW.md](WORKFLOW.md) - End-to-end usage workflows
