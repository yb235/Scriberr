# Database Schema & Models

## Overview

Scriberr uses **SQLite** as its database with **GORM** as the ORM (Object-Relational Mapping) layer. The database stores all application data including users, transcription jobs, chat sessions, and configuration.

**Database File**: `data/scriberr.db` (default location)

## Technology Stack

- **Engine**: SQLite 3
- **ORM**: GORM (Go ORM library)
- **Migration**: Auto-migration via GORM
- **Driver**: github.com/glebarez/sqlite (pure Go SQLite driver)

## Core Models

### User Model

**File**: `internal/models/auth.go`

```go
type User struct {
    ID           string    `json:"id" gorm:"primaryKey;type:varchar(36)"`
    Username     string    `json:"username" gorm:"uniqueIndex;not null"`
    PasswordHash string    `json:"-" gorm:"not null"`
    CreatedAt    time.Time `json:"created_at" gorm:"autoCreateTime"`
    UpdatedAt    time.Time `json:"updated_at" gorm:"autoUpdateTime"`
}
```

**SQL Schema**:
```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    username TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at DATETIME,
    updated_at DATETIME
);

CREATE UNIQUE INDEX idx_users_username ON users(username);
```

**Fields**:
- `id`: UUID primary key
- `username`: Unique username
- `password_hash`: Bcrypt hashed password (cost 10)
- `created_at`: Account creation timestamp
- `updated_at`: Last update timestamp

**Security**:
- Passwords hashed with bcrypt
- Never exposed in JSON responses (`json:"-"`)
- Username uniqueness enforced at DB level

### TranscriptionJob Model

**File**: `internal/models/transcription.go`

```go
type TranscriptionJob struct {
    ID                    string         `json:"id" gorm:"primaryKey;type:varchar(36)"`
    Title                 *string        `json:"title,omitempty" gorm:"type:text"`
    Status                JobStatus      `json:"status" gorm:"type:varchar(20);not null;default:'pending'"`
    AudioPath             string         `json:"audio_path" gorm:"type:text;not null"`
    Transcript            *string        `json:"transcript,omitempty" gorm:"type:text"`
    Diarization           bool           `json:"diarization" gorm:"type:boolean;default:false"`
    Summary               *string        `json:"summary,omitempty" gorm:"type:text"`
    ErrorMessage          *string        `json:"error_message,omitempty" gorm:"type:text"`
    IsMultiTrack          bool           `json:"is_multi_track" gorm:"type:boolean;default:false"`
    AupFilePath           *string        `json:"aup_file_path,omitempty" gorm:"type:text"`
    MultiTrackFolder      *string        `json:"multi_track_folder,omitempty" gorm:"type:text"`
    MergedAudioPath       *string        `json:"merged_audio_path,omitempty" gorm:"type:text"`
    MergeStatus           string         `json:"merge_status" gorm:"type:varchar(20);default:'none'"`
    MergeError            *string        `json:"merge_error,omitempty" gorm:"type:text"`
    IndividualTranscripts *string        `json:"individual_transcripts,omitempty" gorm:"type:text"`
    CreatedAt             time.Time      `json:"created_at" gorm:"autoCreateTime"`
    UpdatedAt             time.Time      `json:"updated_at" gorm:"autoUpdateTime"`
    
    // Embedded parameters
    Parameters            WhisperXParams `json:"parameters" gorm:"embedded"`
    
    // Relationships
    MultiTrackFiles       []MultiTrackFile `json:"multi_track_files,omitempty" gorm:"foreignKey:TranscriptionJobID"`
}
```

**Job Status Enum**:
```go
type JobStatus string

const (
    StatusUploaded   JobStatus = "uploaded"    // File uploaded, not started
    StatusPending    JobStatus = "pending"     // In queue, waiting
    StatusProcessing JobStatus = "processing"  // Currently transcribing
    StatusCompleted  JobStatus = "completed"   // Finished successfully
    StatusFailed     JobStatus = "failed"      // Error occurred
)
```

**WhisperXParams (Embedded)**:
```go
type WhisperXParams struct {
    // Model configuration
    ModelFamily    string  `json:"model_family" gorm:"type:varchar(20);default:'whisper'"`
    Model          string  `json:"model" gorm:"type:varchar(50);default:'small'"`
    ModelCacheOnly bool    `json:"model_cache_only" gorm:"type:boolean;default:false"`
    ModelDir       *string `json:"model_dir,omitempty" gorm:"type:text"`
    
    // Compute settings
    Device         string  `json:"device" gorm:"type:varchar(20);default:'cpu'"`
    DeviceIndex    int     `json:"device_index" gorm:"type:int;default:0"`
    BatchSize      int     `json:"batch_size" gorm:"type:int;default:8"`
    ComputeType    string  `json:"compute_type" gorm:"type:varchar(20);default:'float32'"`
    Threads        int     `json:"threads" gorm:"type:int;default:0"`
    
    // Output settings
    OutputFormat   string  `json:"output_format" gorm:"type:varchar(20);default:'all'"`
    Verbose        bool    `json:"verbose" gorm:"type:boolean;default:true"`
    
    // Language and task
    Task           string  `json:"task" gorm:"type:varchar(20);default:'transcribe'"`
    Language       *string `json:"language,omitempty" gorm:"type:varchar(10)"`
    
    // Alignment
    AlignModel              *string `json:"align_model,omitempty" gorm:"type:varchar(100)"`
    InterpolateMethod       string  `json:"interpolate_method" gorm:"type:varchar(20);default:'nearest'"`
    NoAlign                 bool    `json:"no_align" gorm:"type:boolean;default:false"`
    ReturnCharAlignments    bool    `json:"return_char_alignments" gorm:"type:boolean;default:false"`
    
    // VAD settings
    VadMethod      string   `json:"vad_method" gorm:"type:varchar(20);default:'pyannote'"`
    VadOnset       *float64 `json:"vad_onset,omitempty" gorm:"type:real"`
    VadOffset      *float64 `json:"vad_offset,omitempty" gorm:"type:real"`
    
    // Diarization
    DiarizationModel string  `json:"diarization_model,omitempty" gorm:"type:varchar(100)"`
    MinSpeakers      *int    `json:"min_speakers,omitempty" gorm:"type:int"`
    MaxSpeakers      *int    `json:"max_speakers,omitempty" gorm:"type:int"`
    NumSpeakers      *int    `json:"num_speakers,omitempty" gorm:"type:int"`
    HFToken          *string `json:"hf_token,omitempty" gorm:"type:text"`
    
    // Additional options
    PrintProgress    bool    `json:"print_progress" gorm:"type:boolean;default:false"`
    MaxLineWidth     *int    `json:"max_line_width,omitempty" gorm:"type:int"`
    MaxLineCount     *int    `json:"max_line_count,omitempty" gorm:"type:int"`
    HighlightWords   bool    `json:"highlight_words" gorm:"type:boolean;default:false"`
}
```

**Indexes**:
```sql
CREATE INDEX idx_transcription_jobs_status ON transcription_jobs(status);
CREATE INDEX idx_transcription_jobs_created_at ON transcription_jobs(created_at);
```

### TranscriptionJobExecution Model

Tracks each execution attempt of a transcription job.

```go
type TranscriptionJobExecution struct {
    ID                     string          `json:"id" gorm:"primaryKey;type:varchar(36)"`
    TranscriptionJobID     string          `json:"transcription_job_id" gorm:"type:varchar(36);not null;index"`
    StartedAt              time.Time       `json:"started_at"`
    CompletedAt            *time.Time      `json:"completed_at,omitempty"`
    ProcessingDuration     *int            `json:"processing_duration_seconds,omitempty"`
    ActualParameters       WhisperXParams  `json:"actual_parameters" gorm:"embedded;embeddedPrefix:param_"`
    Status                 JobStatus       `json:"status" gorm:"type:varchar(20);not null"`
    ErrorMessage           *string         `json:"error_message,omitempty" gorm:"type:text"`
    LogOutput              *string         `json:"log_output,omitempty" gorm:"type:text"`
    
    // Relationship
    TranscriptionJob       *TranscriptionJob `json:"-" gorm:"foreignKey:TranscriptionJobID"`
}
```

**Purpose**: 
- Audit trail of processing attempts
- Store execution logs and errors
- Track processing duration
- Compare parameters across retries

### Note Model

**File**: `internal/models/note.go`

```go
type Note struct {
    ID                   string    `json:"id" gorm:"primaryKey;type:varchar(36)"`
    TranscriptionJobID   string    `json:"transcription_job_id" gorm:"type:varchar(36);not null;index"`
    Content              string    `json:"content" gorm:"type:text;not null"`
    Timestamp            *float64  `json:"timestamp,omitempty" gorm:"type:real"`
    CreatedAt            time.Time `json:"created_at" gorm:"autoCreateTime"`
    UpdatedAt            time.Time `json:"updated_at" gorm:"autoUpdateTime"`
    
    // Relationship
    TranscriptionJob     *TranscriptionJob `json:"-" gorm:"foreignKey:TranscriptionJobID"`
}
```

**SQL Schema**:
```sql
CREATE TABLE notes (
    id TEXT PRIMARY KEY,
    transcription_job_id TEXT NOT NULL,
    content TEXT NOT NULL,
    timestamp REAL,
    created_at DATETIME,
    updated_at DATETIME,
    FOREIGN KEY (transcription_job_id) REFERENCES transcription_jobs(id) ON DELETE CASCADE
);

CREATE INDEX idx_notes_transcription_job_id ON notes(transcription_job_id);
```

**Usage**:
- User adds notes during transcript review
- Optional timestamp links note to audio position
- Cascade delete when parent job deleted

### APIKey Model

```go
type APIKey struct {
    ID        string     `json:"id" gorm:"primaryKey;type:varchar(36)"`
    UserID    string     `json:"user_id" gorm:"type:varchar(36);not null;index"`
    KeyHash   string     `json:"-" gorm:"type:varchar(64);uniqueIndex;not null"`
    Name      string     `json:"name" gorm:"type:varchar(100);not null"`
    CreatedAt time.Time  `json:"created_at" gorm:"autoCreateTime"`
    ExpiresAt *time.Time `json:"expires_at,omitempty"`
    LastUsed  *time.Time `json:"last_used,omitempty"`
    
    // Relationship
    User      *User      `json:"-" gorm:"foreignKey:UserID"`
}
```

**Security**:
- Keys are SHA-256 hashed before storage
- Original key shown only once at creation
- Expiration optional but recommended
- Last used timestamp for auditing

**Indexes**:
```sql
CREATE UNIQUE INDEX idx_api_keys_key_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
```

### Profile Model

Saves reusable transcription configurations.

```go
type Profile struct {
    ID          string         `json:"id" gorm:"primaryKey;type:varchar(36)"`
    UserID      string         `json:"user_id" gorm:"type:varchar(36);not null;index"`
    Name        string         `json:"name" gorm:"type:varchar(100);not null"`
    Description *string        `json:"description,omitempty" gorm:"type:text"`
    IsDefault   bool           `json:"is_default" gorm:"type:boolean;default:false"`
    Parameters  WhisperXParams `json:"parameters" gorm:"embedded"`
    CreatedAt   time.Time      `json:"created_at" gorm:"autoCreateTime"`
    UpdatedAt   time.Time      `json:"updated_at" gorm:"autoUpdateTime"`
    
    // Relationship
    User        *User          `json:"-" gorm:"foreignKey:UserID"`
}
```

**Usage**:
- Save frequently used parameter sets
- Quick selection for new jobs
- One default profile per user

### ChatSession Model

```go
type ChatSession struct {
    ID                   string             `json:"id" gorm:"primaryKey;type:varchar(36)"`
    TranscriptionJobID   string             `json:"transcription_job_id" gorm:"type:varchar(36);not null;index"`
    Title                *string            `json:"title,omitempty" gorm:"type:varchar(200)"`
    CreatedAt            time.Time          `json:"created_at" gorm:"autoCreateTime"`
    UpdatedAt            time.Time          `json:"updated_at" gorm:"autoUpdateTime"`
    
    // Relationships
    TranscriptionJob     *TranscriptionJob  `json:"-" gorm:"foreignKey:TranscriptionJobID"`
    Messages             []ChatMessage      `json:"messages,omitempty" gorm:"foreignKey:ChatSessionID"`
}
```

### ChatMessage Model

```go
type ChatMessage struct {
    ID            string       `json:"id" gorm:"primaryKey;type:varchar(36)"`
    ChatSessionID string       `json:"chat_session_id" gorm:"type:varchar(36);not null;index"`
    Role          string       `json:"role" gorm:"type:varchar(20);not null"`
    Content       string       `json:"content" gorm:"type:text;not null"`
    Model         *string      `json:"model,omitempty" gorm:"type:varchar(100)"`
    Temperature   *float64     `json:"temperature,omitempty" gorm:"type:real"`
    CreatedAt     time.Time    `json:"created_at" gorm:"autoCreateTime"`
    
    // Relationship
    ChatSession   *ChatSession `json:"-" gorm:"foreignKey:ChatSessionID"`
}
```

**Roles**: `user`, `assistant`, `system`

**Cascade Delete**: Messages deleted when session deleted

### SummaryTemplate Model

```go
type SummaryTemplate struct {
    ID            string    `json:"id" gorm:"primaryKey;type:varchar(36)"`
    UserID        string    `json:"user_id" gorm:"type:varchar(36);not null;index"`
    Name          string    `json:"name" gorm:"type:varchar(100);not null"`
    Description   *string   `json:"description,omitempty" gorm:"type:text"`
    SystemPrompt  string    `json:"system_prompt" gorm:"type:text;not null"`
    UserPrompt    string    `json:"user_prompt" gorm:"type:text;not null"`
    Model         *string   `json:"model,omitempty" gorm:"type:varchar(100)"`
    Temperature   *float64  `json:"temperature,omitempty" gorm:"type:real"`
    CreatedAt     time.Time `json:"created_at" gorm:"autoCreateTime"`
    UpdatedAt     time.Time `json:"updated_at" gorm:"autoUpdateTime"`
    
    // Relationship
    User          *User     `json:"-" gorm:"foreignKey:UserID"`
}
```

**Purpose**:
- Reusable prompt templates for summaries
- Customize system and user prompts
- Save model preferences

### UserSettings Model

```go
type UserSettings struct {
    UserID              string    `json:"user_id" gorm:"primaryKey;type:varchar(36)"`
    DefaultProfileID    *string   `json:"default_profile_id,omitempty" gorm:"type:varchar(36)"`
    LLMProvider         string    `json:"llm_provider" gorm:"type:varchar(20);default:'openai'"`
    OpenAIAPIKey        *string   `json:"openai_api_key,omitempty" gorm:"type:text"`
    OpenAIBaseURL       *string   `json:"openai_base_url,omitempty" gorm:"type:text"`
    OllamaBaseURL       *string   `json:"ollama_base_url,omitempty" gorm:"type:text"`
    DefaultModel        *string   `json:"default_model,omitempty" gorm:"type:varchar(100)"`
    UpdatedAt           time.Time `json:"updated_at" gorm:"autoUpdateTime"`
    
    // Relationship
    User                *User     `json:"-" gorm:"foreignKey:UserID"`
}
```

**LLM Provider**: `openai` or `ollama`

### MultiTrackFile Model

```go
type MultiTrackFile struct {
    ID                   string             `json:"id" gorm:"primaryKey;type:varchar(36)"`
    TranscriptionJobID   string             `json:"transcription_job_id" gorm:"type:varchar(36);not null;index"`
    TrackName            string             `json:"track_name" gorm:"type:varchar(200);not null"`
    FilePath             string             `json:"file_path" gorm:"type:text;not null"`
    TranscriptPath       *string            `json:"transcript_path,omitempty" gorm:"type:text"`
    Status               string             `json:"status" gorm:"type:varchar(20);default:'pending'"`
    CreatedAt            time.Time          `json:"created_at" gorm:"autoCreateTime"`
    
    // Relationship
    TranscriptionJob     *TranscriptionJob  `json:"-" gorm:"foreignKey:TranscriptionJobID"`
}
```

**Purpose**: Track individual tracks in multi-track Audacity projects

## Database Initialization

**File**: `internal/database/database.go`

```go
func Initialize(dbPath string) error {
    var err error
    DB, err = gorm.Open(sqlite.Open(dbPath), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    })
    if err != nil {
        return fmt.Errorf("failed to connect to database: %w", err)
    }

    // Auto-migrate all models
    if err := DB.AutoMigrate(
        &models.User{},
        &models.TranscriptionJob{},
        &models.TranscriptionJobExecution{},
        &models.Note{},
        &models.APIKey{},
        &models.Profile{},
        &models.ChatSession{},
        &models.ChatMessage{},
        &models.SummaryTemplate{},
        &models.UserSettings{},
        &models.MultiTrackFile{},
    ); err != nil {
        return fmt.Errorf("failed to migrate database: %w", err)
    }

    return nil
}
```

**Auto-Migration**:
- Creates tables if they don't exist
- Adds missing columns
- Creates indexes
- Does NOT delete columns or data

## Common Queries

### User Queries

```go
// Find user by username
var user models.User
db.Where("username = ?", username).First(&user)

// Create user
user := models.User{
    ID:           uuid.New().String(),
    Username:     username,
    PasswordHash: hashedPassword,
}
db.Create(&user)
```

### Transcription Queries

```go
// List jobs with pagination
var jobs []models.TranscriptionJob
db.Order("created_at DESC").
   Limit(perPage).
   Offset((page - 1) * perPage).
   Find(&jobs)

// Get job with relationships
var job models.TranscriptionJob
db.Preload("MultiTrackFiles").
   Where("id = ?", jobID).
   First(&job)

// Update job status
db.Model(&models.TranscriptionJob{}).
   Where("id = ?", jobID).
   Update("status", "completed")
```

### Note Queries

```go
// List notes for transcription
var notes []models.Note
db.Where("transcription_job_id = ?", jobID).
   Order("timestamp ASC").
   Find(&notes)

// Create note
note := models.Note{
    ID:                 uuid.New().String(),
    TranscriptionJobID: jobID,
    Content:            content,
    Timestamp:          &timestamp,
}
db.Create(&note)
```

### Chat Queries

```go
// Get session with messages
var session models.ChatSession
db.Preload("Messages").
   Where("id = ?", sessionID).
   First(&session)

// Add message to session
message := models.ChatMessage{
    ID:            uuid.New().String(),
    ChatSessionID: sessionID,
    Role:          "user",
    Content:       content,
}
db.Create(&message)
```

## Data Relationships

```
User (1) ──┬── (N) APIKey
           ├── (N) Profile
           ├── (N) SummaryTemplate
           └── (1) UserSettings

TranscriptionJob (1) ──┬── (N) Note
                       ├── (N) ChatSession
                       ├── (N) TranscriptionJobExecution
                       └── (N) MultiTrackFile

ChatSession (1) ──── (N) ChatMessage
```

## Indexes & Performance

### Primary Keys
All models use UUID strings as primary keys:
- Globally unique
- URL-safe
- Non-sequential (security)

### Indexes
- Foreign keys automatically indexed
- Unique constraints on usernames, API key hashes
- Status and timestamp indexes for filtering

### Query Optimization
```go
// Good: Use indexes
db.Where("status = ?", "completed").Find(&jobs)

// Bad: Full table scan
db.Where("created_at > ?", time.Now().Add(-24*time.Hour)).Find(&jobs)

// Good: Index on created_at
db.Where("created_at > ?", time.Now().Add(-24*time.Hour)).
   Order("created_at DESC").
   Find(&jobs)
```

## Migrations

### Manual Migrations

If you need to modify the schema beyond what auto-migrate supports:

```go
// Add column
db.Exec("ALTER TABLE transcription_jobs ADD COLUMN new_field TEXT")

// Create index
db.Exec("CREATE INDEX idx_custom ON table_name(column)")

// Update data
db.Model(&models.TranscriptionJob{}).
   Where("status = ?", "old_status").
   Update("status", "new_status")
```

### Backup

```bash
# SQLite backup
cp data/scriberr.db data/scriberr.db.backup

# Or use SQLite backup command
sqlite3 data/scriberr.db ".backup data/backup.db"
```

## Database Maintenance

### Vacuum

Reclaim space after deletes:
```sql
VACUUM;
```

### Analyze

Update query planner statistics:
```sql
ANALYZE;
```

### Check Integrity

```sql
PRAGMA integrity_check;
```

## Constraints

### Foreign Keys

SQLite foreign keys must be enabled:
```go
db.Exec("PRAGMA foreign_keys = ON")
```

### Cascade Deletes

Configured at model level:
```go
gorm:"constraint:OnDelete:CASCADE"
```

## Best Practices

1. **Use Transactions**: For multiple related operations
2. **Preload Relationships**: Avoid N+1 queries
3. **Index Foreign Keys**: Already done automatically
4. **Limit Results**: Always paginate large queries
5. **Use Where Clauses**: Never load entire tables
6. **Soft Deletes**: Consider adding deleted_at if needed
7. **Backup Regularly**: SQLite databases are single files

## Future Considerations

### Scaling Beyond SQLite

When you outgrow SQLite:

1. **PostgreSQL**: Full SQL database
2. **Separate Read/Write**: Read replicas
3. **Partitioning**: Partition by date
4. **Archive Old Data**: Move completed jobs to archive

### Current Limitations

- **Concurrent Writes**: SQLite locks on writes
- **Single Server**: Not distributed
- **File Size**: Practical limit ~100GB
- **Full-Text Search**: Limited compared to PostgreSQL

For most self-hosted deployments, SQLite is more than sufficient.
