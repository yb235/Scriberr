# Scriberr Architecture

## System Overview

Scriberr follows a **three-tier architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend (React)                        │
│  - User Interface Components                                │
│  - State Management (Context API)                           │
│  - API Client                                               │
└─────────────────────┬───────────────────────────────────────┘
                      │ HTTP/REST API
┌─────────────────────▼───────────────────────────────────────┐
│                   Backend (Go)                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │ API Layer  │  │ Auth       │  │ Queue      │           │
│  │ (Handlers) │  │ Service    │  │ System     │           │
│  └────────────┘  └────────────┘  └────────────┘           │
│  ┌────────────────────────────────────────────┐            │
│  │       Transcription Service                │            │
│  │  (Orchestrates Python Adapters)            │            │
│  └────────────────────────────────────────────┘            │
│  ┌────────────────────────────────────────────┐            │
│  │       Database (SQLite + GORM)             │            │
│  └────────────────────────────────────────────┘            │
└─────────────────────┬───────────────────────────────────────┘
                      │ Python Process Execution
┌─────────────────────▼───────────────────────────────────────┐
│              Processing Layer (Python)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ WhisperX │  │ Pyannote │  │ NVIDIA   │                 │
│  │ Adapter  │  │ Adapter  │  │ Adapters │                 │
│  └──────────┘  └──────────┘  └──────────┘                 │
│  - Isolated Python environments per model family           │
│  - Model downloading and caching                           │
│  - Audio preprocessing and transcription                   │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Frontend Layer (React)

**Location**: `web/frontend/`

#### Component Structure
```
web/frontend/src/
├── App.tsx                 # Main application component
├── main.tsx               # Application entry point
├── pages/                 # Page-level components
│   ├── Login.tsx
│   ├── Register.tsx
│   ├── Settings.tsx
│   └── ChatPage.tsx
├── components/            # Reusable UI components
│   ├── AudioRecorder.tsx
│   ├── TranscriptionConfigDialog.tsx
│   ├── ProfilesTable.tsx
│   ├── LLMSettings.tsx
│   └── SummaryTemplatesTable.tsx
├── contexts/              # React Context providers
│   └── AuthContext.tsx
├── lib/                   # Utilities and helpers
└── types/                 # TypeScript type definitions
```

#### Key Responsibilities
- **User Interface**: Renders all UI elements using shadcn/ui + Tailwind
- **State Management**: React Context for auth, settings, transcriptions
- **API Communication**: Fetch API calls to backend endpoints
- **Audio Playback**: HTML5 audio with custom controls
- **Real-time Updates**: Polling for job status, streaming chat responses

#### Build Process
1. Vite bundles React app into static files (HTML, CSS, JS)
2. Build script copies `web/frontend/dist` to `internal/web/dist`
3. Go's `embed` package includes files in compiled binary
4. Backend serves files from embedded filesystem

### 2. Backend Layer (Go)

**Location**: `internal/` and `cmd/server/`

#### Package Structure
```
internal/
├── api/                   # HTTP handlers and routing
│   ├── router.go         # Route definitions
│   ├── handlers.go       # Transcription endpoints
│   ├── chat_handlers.go  # Chat/LLM endpoints
│   ├── notes_handlers.go # Note management
│   └── summary_handlers.go # Summary templates
├── auth/                  # Authentication service
│   └── auth.go           # JWT + API key auth
├── database/              # Database connection
│   └── database.go       # GORM setup
├── models/                # Database models
│   ├── transcription.go  # Job models
│   ├── auth.go           # User models
│   └── note.go           # Note models
├── queue/                 # Job queue system
│   └── queue.go          # Worker pool implementation
├── transcription/         # Transcription orchestration
│   ├── unified_service.go      # Main service
│   ├── adapters/              # Model adapters
│   │   ├── base_adapter.go
│   │   ├── whisperx_adapter.go
│   │   ├── pyannote_adapter.go
│   │   ├── parakeet_adapter.go
│   │   ├── canary_adapter.go
│   │   └── sortformer_adapter.go
│   ├── registry/              # Model registry
│   │   └── registry.go
│   ├── pipeline/              # Processing pipeline
│   │   └── pipeline.go
│   └── interfaces/            # Adapter interfaces
│       └── interfaces.go
├── llm/                   # LLM integration
│   ├── service.go        # LLM interface
│   ├── openai.go         # OpenAI implementation
│   └── ollama.go         # Ollama implementation
├── web/                   # Static file serving
│   └── dist/             # Embedded frontend (build-time)
└── config/                # Configuration loading
    └── config.go
```

#### Key Responsibilities

**API Layer** (`internal/api/`)
- Request handling and validation
- Response formatting
- Middleware (auth, CORS, compression)
- Swagger documentation annotations

**Authentication** (`internal/auth/`)
- JWT token generation and validation
- API key creation and verification
- Password hashing (bcrypt)
- Session management

**Queue System** (`internal/queue/`)
- Auto-scaling worker pool
- Job distribution and execution
- Process tracking and cancellation
- Resource management

**Transcription Service** (`internal/transcription/`)
- Adapter pattern for model abstraction
- Python environment management
- Job lifecycle orchestration
- Model registry and discovery

**Database** (`internal/database/` + `internal/models/`)
- SQLite connection via GORM
- Schema migrations
- Model definitions
- Query builders

**LLM Integration** (`internal/llm/`)
- Provider-agnostic interface
- OpenAI API client
- Ollama local model client
- Streaming support

### 3. Processing Layer (Python)

**Location**: Python environments managed dynamically

#### Environment Isolation

Scriberr creates separate Python environments for different model families:

```
data/
├── whisperx-env/           # WhisperX + Pyannote
│   ├── pyproject.toml
│   └── .venv/
├── nvidia-env/             # NVIDIA Parakeet, Canary, Sortformer
│   ├── pyproject.toml
│   └── .venv/
└── models/                 # Downloaded model files
    ├── whisper/
    ├── pyannote/
    └── nvidia/
```

#### Adapter Pattern

Each model implements the `TranscriptionAdapter` or `DiarizationAdapter` interface:

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
- Easy to add new models
- Consistent configuration across models
- Independent environment management
- Runtime model switching

#### Model Adapters

**WhisperX Adapter** (`whisperx_adapter.go`)
- OpenAI Whisper models (tiny → large-v3)
- Word-level timestamps
- Language detection
- VAD (Voice Activity Detection)
- Environment: `whisperx-env`

**Pyannote Adapter** (`pyannote_adapter.go`)
- Speaker diarization
- Requires Hugging Face token
- Multiple model versions (3.0, 3.1)
- Environment: `whisperx-env` (shared)

**NVIDIA Parakeet** (`parakeet_adapter.go`)
- Fast ASR model
- GPU-optimized
- Environment: `nvidia-env`

**NVIDIA Canary** (`canary_adapter.go`)
- Multilingual support
- Code-switching capable
- Environment: `nvidia-env`

**NVIDIA Sortformer** (`sortformer_adapter.go`)
- Streaming diarization
- Multi-speaker support
- Environment: `nvidia-env`

#### Python Execution Flow

1. **Environment Check**: Verify Python environment exists
2. **Model Download**: Download models if not cached
3. **Script Generation**: Create Python script with parameters
4. **Process Launch**: Execute Python with `uv run`
5. **Output Capture**: Read JSON results from stdout
6. **Error Handling**: Parse stderr for debugging

### 4. Database Schema

**Engine**: SQLite with GORM ORM

#### Core Tables

**users**
```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at DATETIME,
    updated_at DATETIME
);
```

**transcription_jobs**
```sql
CREATE TABLE transcription_jobs (
    id TEXT PRIMARY KEY,
    title TEXT,
    status TEXT NOT NULL,
    audio_path TEXT NOT NULL,
    transcript TEXT,
    diarization BOOLEAN DEFAULT FALSE,
    summary TEXT,
    error_message TEXT,
    is_multi_track BOOLEAN DEFAULT FALSE,
    -- WhisperX parameters (embedded)
    model_family TEXT DEFAULT 'whisper',
    model TEXT DEFAULT 'small',
    device TEXT DEFAULT 'cpu',
    batch_size INTEGER DEFAULT 8,
    -- ... more parameters
    created_at DATETIME,
    updated_at DATETIME
);
```

**transcription_job_executions**
- Tracks each execution attempt
- Stores actual parameters used
- Processing duration and timestamps
- Execution-specific errors

**notes**
```sql
CREATE TABLE notes (
    id TEXT PRIMARY KEY,
    transcription_job_id TEXT NOT NULL,
    content TEXT NOT NULL,
    timestamp REAL,
    created_at DATETIME,
    FOREIGN KEY (transcription_job_id) REFERENCES transcription_jobs(id)
);
```

**chat_sessions** & **chat_messages**
- Chat history per transcription
- Message role (user/assistant)
- Model and temperature used
- Auto-generated titles

**summary_templates**
- Reusable prompt templates
- System and user prompts
- Model preferences

**api_keys**
- Hashed API keys
- Expiration dates
- User associations

**profiles**
- Saved transcription configurations
- Default profile per user
- Reusable parameter sets

### 5. Job Queue System

**Location**: `internal/queue/queue.go`

#### Architecture

```
                    ┌──────────────┐
                    │  Job Channel │
                    │  (buffered)  │
                    └───────┬──────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
   │ Worker  │        │ Worker  │        │ Worker  │
   │  Pool   │        │  Pool   │        │  Pool   │
   └─────────┘        └─────────┘        └─────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                    ┌───────▼────────┐
                    │   Processor    │
                    │  (calls model  │
                    │   adapters)    │
                    └────────────────┘
```

#### Features

**Auto-Scaling Worker Pool**
- Starts with minimum workers (1-2)
- Scales up based on queue depth
- Scales down when idle
- Respects system CPU count

**Job Tracking**
- Track running jobs with context cancellation
- Store process handles for termination
- Graceful shutdown support
- Status updates in database

**Process Management**
- OS-level process tracking
- Cross-platform kill support (Linux, macOS, Windows)
- Multi-track job termination
- Cleanup of temporary files

## Data Flow

### Transcription Request Flow

```
User Upload Audio
    │
    ▼
Frontend: POST /api/v1/transcription/upload
    │
    ▼
Backend: Upload Handler
    │ - Save file to disk
    │ - Create database record
    │ - Return job ID
    ▼
Frontend: POST /api/v1/transcription/:id/start
    │
    ▼
Backend: Start Handler
    │ - Validate parameters
    │ - Enqueue job
    │ - Return accepted status
    ▼
Queue: Worker picks up job
    │
    ▼
Unified Service: ProcessJob()
    │ - Load job from DB
    │ - Select adapter (WhisperX, Parakeet, etc.)
    │ - Call adapter.PrepareEnvironment()
    │ - Call adapter.Transcribe()
    │
    ▼
Python Adapter: Execute transcription
    │ - Setup Python env (if needed)
    │ - Download model (if needed)
    │ - Run transcription script
    │ - Parse JSON output
    │
    ▼
Unified Service: Save results
    │ - Update job status
    │ - Save transcript
    │ - Create execution record
    │
    ▼
Frontend: Poll GET /api/v1/transcription/:id/status
    │
    ▼
User sees completed transcript
```

### Chat Request Flow

```
User sends message
    │
    ▼
Frontend: POST /api/v1/chat/sessions/:id/messages
    │
    ▼
Backend: Chat Handler
    │ - Load transcription
    │ - Load chat history
    │ - Prepare context
    │
    ▼
LLM Service: ChatCompletionStream()
    │
    ├─ OpenAI: Call API
    │  └─ Stream responses
    │
    └─ Ollama: Call local server
       └─ Stream responses
    │
    ▼
Backend: Stream to client
    │ - Save message to DB
    │ - Return SSE stream
    │
    ▼
Frontend: Display streaming response
```

## Communication Patterns

### Frontend ↔ Backend

**Authentication**
- JWT tokens in `Authorization` header
- API keys in `X-API-Key` header
- Refresh token rotation

**File Upload**
- Multipart form data
- Chunked transfer for large files
- Compression disabled for uploads

**Status Polling**
- Frontend polls every 2-5 seconds during processing
- Backend returns job status + progress
- Stops polling when job completes/fails

**Streaming**
- Server-Sent Events (SSE) for chat responses
- Event stream with incremental tokens
- Error handling via event stream

### Backend ↔ Python

**Execution Model**
- Go spawns Python processes via `exec.Command`
- Communication via stdin/stdout/stderr
- JSON serialization for data exchange

**Environment Management**
- Go checks for Python environment
- Creates/updates using `uv` package manager
- Caches environment across requests

**Error Handling**
- Python writes errors to stderr
- Go captures and logs stderr
- Exit codes indicate success/failure

## Security Architecture

### Authentication & Authorization

**JWT Tokens**
- HS256 signing algorithm
- 24-hour expiration
- Refresh token flow
- User ID in claims

**API Keys**
- SHA-256 hashed before storage
- Optional expiration
- User-scoped permissions
- Revocable

**Password Security**
- Bcrypt hashing (cost 10)
- No plaintext storage
- Rate limiting on login (middleware-ready)

### Data Protection

**Database**
- SQLite file permissions (0600)
- No sensitive data in logs
- Prepared statements (SQL injection protection)

**File Storage**
- Isolated upload directories
- Path traversal prevention
- Unique filenames (UUID)

**API Security**
- CORS configuration
- Request size limits
- Input validation
- SQL injection protection (ORM)

## Performance Optimizations

### Backend

**Compression Middleware**
- Gzip compression for API responses
- Disabled for file uploads/downloads
- Configurable compression level

**Database**
- Connection pooling via GORM
- Indexes on foreign keys
- Lazy loading relationships

**Worker Pool**
- Auto-scaling based on load
- Prevents resource exhaustion
- Graceful degradation

### Frontend

**Code Splitting**
- Route-based code splitting
- Lazy loading of heavy components
- Tree shaking unused code

**Asset Optimization**
- Vite builds minified bundles
- CSS purging via Tailwind
- Static asset caching

**API Efficiency**
- Debounced polling
- Request cancellation
- Response caching where appropriate

### Python

**Model Caching**
- Downloaded models reused
- Environment reused across jobs
- Warm start for faster processing

**Batch Processing**
- Configurable batch sizes
- GPU memory optimization
- Parallel processing where possible

## Scalability Considerations

### Current Limitations

- **Single Node**: Designed for single-server deployment
- **SQLite**: Not suitable for high-concurrency writes
- **Local Storage**: Files stored on local filesystem
- **Sequential Processing**: Jobs processed one-by-one per worker

### Future Scaling Paths

- **Database**: Migrate to PostgreSQL for multi-server
- **Storage**: S3/MinIO for distributed file storage
- **Queue**: Redis or RabbitMQ for distributed queue
- **Workers**: Separate worker nodes
- **Load Balancing**: Multiple API servers

## Extension Points

### Adding New Models

1. Create adapter implementing `TranscriptionAdapter` or `DiarizationAdapter`
2. Register in `init()` function
3. Define parameter schema
4. Implement environment setup
5. Create Python script template

### Adding New LLM Providers

1. Implement `llm.Service` interface
2. Add to `llm/` package
3. Update settings UI
4. Add provider configuration

### Custom Preprocessing

1. Implement `Preprocessor` interface
2. Register with pipeline
3. Chain before transcription

### Custom Postprocessing

1. Implement `Postprocessor` interface
2. Register with pipeline
3. Chain after transcription

## Deployment Architecture

### Single Binary

```
scriberr (executable)
├── Embedded Frontend (React build)
├── API Server (Gin)
├── Database (SQLite)
├── Queue Workers
└── Python Orchestration
```

### Docker Container

```
Container Image
├── Base: Python 3.11 slim
├── System: ffmpeg, git, build tools
├── Go Binary (scriberr)
├── UV package manager
├── Entrypoint script (user management)
└── Volumes: /app/data
```

### Data Persistence

```
data/
├── scriberr.db              # SQLite database
├── uploads/                 # Audio files
│   └── {job-id}.{ext}
├── transcripts/             # JSON outputs
│   └── {job-id}.json
├── whisperx-env/            # Python env
├── nvidia-env/              # Python env
└── models/                  # Cached models
```

## Conclusion

Scriberr's architecture prioritizes:
- **Simplicity**: Single binary, minimal dependencies
- **Flexibility**: Adapter pattern for extensibility
- **Privacy**: Local processing, no cloud dependency
- **Performance**: Auto-scaling, caching, optimization
- **Maintainability**: Clear separation of concerns, standard patterns

This design enables easy deployment, modification, and scaling while maintaining a great user experience.
