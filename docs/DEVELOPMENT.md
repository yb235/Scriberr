# Development Guide

## Getting Started

This guide covers setting up a local development environment for Scriberr.

## Prerequisites

### Required
- **Go**: 1.24 or later
- **Node.js**: 20.x or later
- **npm**: 10.x or later
- **Python**: 3.11 or later
- **Git**: For cloning the repository

### Optional
- **UV**: Fast Python package manager (auto-installed if missing)
- **Docker**: For containerized development
- **CUDA**: For GPU-accelerated transcription

## Installation Steps

### 1. Clone Repository

```bash
git clone https://github.com/rishikanthc/Scriberr.git
cd Scriberr
```

### 2. Install Dependencies

**Backend (Go)**:
```bash
# Download Go modules
go mod download

# Verify installation
go version
```

**Frontend (React)**:
```bash
cd web/frontend
npm install
cd ../..
```

### 3. Environment Configuration

Create `.env` file in project root (optional - defaults provided):

```bash
# Server configuration
HOST=localhost
PORT=8080

# Database
DATABASE_PATH=./data/scriberr.db

# File storage
UPLOAD_DIR=./data/uploads

# Python environment
WHISPERX_ENV=./whisperx-env
NVIDIA_ENV=./nvidia-env

# JWT secret (generate with: openssl rand -base64 32)
JWT_SECRET=your-secret-key-here

# Log level (debug, info, warn, error)
LOG_LEVEL=info

# Worker configuration (optional)
QUEUE_WORKERS=2
```

### 4. Initialize Database

The database is auto-initialized on first run. To manually initialize:

```bash
go run cmd/server/main.go
# Ctrl+C to stop after initialization
```

This creates `data/scriberr.db` and runs migrations.

## Development Workflow

### Running the Backend

**Option 1: Go run (development)**
```bash
go run cmd/server/main.go
```

**Option 2: Build and run**
```bash
go build -o scriberr cmd/server/main.go
./scriberr
```

**Option 3: Air (hot reload)**
```bash
# Install air
go install github.com/cosmtrek/air@latest

# Run with auto-reload
air
```

Backend will start on `http://localhost:8080`

### Running the Frontend

**Development mode** (recommended):
```bash
cd web/frontend
npm run dev
```

Frontend dev server starts on `http://localhost:5173`

**Important**: When running frontend in dev mode:
- Backend must be running on port 8080
- Frontend proxies API requests to backend
- Hot Module Replacement (HMR) enabled

**Production build**:
```bash
cd web/frontend
npm run build
```

Outputs to `web/frontend/dist/`

### Full Build (Embedded Frontend)

To build a single binary with embedded frontend:

```bash
chmod +x ./build.sh
./build.sh
```

This script:
1. Cleans old builds
2. Builds React app (`npm run build`)
3. Copies dist to `internal/web/dist`
4. Builds Go binary with embedded assets
5. Outputs `./scriberr` executable

## Project Structure

```
Scriberr/
├── cmd/
│   └── server/
│       └── main.go           # Application entry point
│
├── internal/                  # Private application code
│   ├── api/                  # HTTP handlers
│   │   ├── router.go
│   │   ├── handlers.go
│   │   ├── chat_handlers.go
│   │   └── ...
│   ├── auth/                 # Authentication service
│   ├── config/               # Configuration loading
│   ├── database/             # Database connection
│   ├── models/               # Data models
│   ├── queue/                # Job queue
│   ├── transcription/        # Transcription service
│   │   ├── adapters/        # Model adapters
│   │   ├── registry/        # Model registry
│   │   ├── pipeline/        # Processing pipeline
│   │   └── interfaces/      # Interfaces
│   ├── llm/                  # LLM integration
│   └── web/                  # Embedded frontend
│
├── web/
│   ├── frontend/             # React application
│   │   ├── src/
│   │   ├── public/
│   │   ├── vite.config.ts
│   │   └── package.json
│   └── landing/              # Marketing site
│
├── api-docs/                 # Swagger documentation
├── docs/                     # Comprehensive documentation
├── tests/                    # Tests
├── .env.example              # Example environment config
├── go.mod                    # Go dependencies
├── go.sum                    # Go dependency checksums
├── build.sh                  # Build script
└── README.md
```

## Coding Standards

### Go Code

**Formatting**:
```bash
# Format all Go files
go fmt ./...

# Vet code for issues
go vet ./...
```

**Linting** (install golangci-lint):
```bash
# Install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Run
golangci-lint run
```

**Style Guidelines**:
- Follow standard Go conventions
- Use meaningful variable names
- Add comments for exported functions
- Keep functions small and focused
- Handle errors explicitly

**Example**:
```go
// GetTranscriptionJob retrieves a transcription job by ID
func GetTranscriptionJob(jobID string) (*models.TranscriptionJob, error) {
    var job models.TranscriptionJob
    
    if err := database.DB.Where("id = ?", jobID).First(&job).Error; err != nil {
        return nil, fmt.Errorf("failed to get job: %w", err)
    }
    
    return &job, nil
}
```

### TypeScript/React Code

**Linting**:
```bash
cd web/frontend
npm run lint
```

**Formatting** (if Prettier configured):
```bash
npm run format
```

**Style Guidelines**:
- Use TypeScript for type safety
- Functional components with hooks
- Descriptive component names
- Extract reusable logic to hooks
- Props interfaces for all components

**Example**:
```tsx
interface TranscriptViewProps {
  jobId: string;
  onComplete?: () => void;
}

export function TranscriptView({ jobId, onComplete }: TranscriptViewProps) {
  const [transcript, setTranscript] = useState<Transcript | null>(null);
  
  useEffect(() => {
    fetchTranscript(jobId).then(setTranscript);
  }, [jobId]);
  
  return (
    <div className="transcript-view">
      {/* Component JSX */}
    </div>
  );
}
```

## Testing

### Backend Tests

**Run all tests**:
```bash
go test ./...
```

**Run specific package**:
```bash
go test ./internal/transcription
```

**With coverage**:
```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

**Example Test**:
```go
func TestWhisperXAdapter_Transcribe(t *testing.T) {
    adapter := NewWhisperXAdapter()
    
    params := TranscriptionParams{
        AudioPath: "test.mp3",
        Model:     "small",
        Language:  "en",
    }
    
    result, err := adapter.Transcribe(context.Background(), params)
    
    assert.NoError(t, err)
    assert.NotNil(t, result)
    assert.Greater(t, len(result.Segments), 0)
}
```

### Frontend Tests

```bash
cd web/frontend
npm run test
```

**Example Test**:
```tsx
import { render, screen } from '@testing-library/react';
import { Button } from './components/Button';

test('button renders with text', () => {
  render(<Button>Click me</Button>);
  expect(screen.getByText('Click me')).toBeInTheDocument();
});
```

## Debugging

### Backend Debugging

**VS Code launch.json**:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Scriberr",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/cmd/server/main.go",
      "env": {
        "LOG_LEVEL": "debug"
      }
    }
  ]
}
```

**Delve CLI**:
```bash
# Install
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug
dlv debug cmd/server/main.go
```

**Logging**:
```go
import "scriberr/pkg/logger"

logger.Debug("Detailed debug info", "key", value)
logger.Info("General information", "user", username)
logger.Warn("Warning message", "error", err)
logger.Error("Error occurred", "error", err)
```

### Frontend Debugging

**Browser DevTools**:
- React DevTools extension
- Console for logs
- Network tab for API calls
- Sources tab for breakpoints

**Console Logging**:
```tsx
console.log('State:', state);
console.error('Error:', error);
console.debug('Debug info:', data);
```

## Adding New Features

### Adding a New API Endpoint

1. **Define Handler** (`internal/api/handlers.go`):
```go
// @Summary Get transcription details
// @Tags transcription
// @Param id path string true "Job ID"
// @Success 200 {object} models.TranscriptionJob
// @Router /transcription/{id} [get]
// @Security BearerAuth
func (h *Handler) GetJobByID(c *gin.Context) {
    jobID := c.Param("id")
    
    var job models.TranscriptionJob
    if err := database.DB.Where("id = ?", jobID).First(&job).Error; err != nil {
        c.JSON(404, gin.H{"error": "Job not found"})
        return
    }
    
    c.JSON(200, job)
}
```

2. **Add Route** (`internal/api/router.go`):
```go
transcription.GET("/:id", handler.GetJobByID)
```

3. **Update Swagger**:
```bash
swag init -g cmd/server/main.go -o api-docs
```

4. **Test**:
```bash
curl http://localhost:8080/api/v1/transcription/job_123 \
  -H "Authorization: Bearer $TOKEN"
```

### Adding a New Model Adapter

1. **Create Adapter** (`internal/transcription/adapters/my_adapter.go`):
```go
type MyAdapter struct {
    *BaseAdapter
}

func NewMyAdapter() *MyAdapter {
    capabilities := interfaces.ModelCapabilities{
        ModelID:     "my-model",
        ModelFamily: "my-family",
        DisplayName: "My Model",
        // ... more capabilities
    }
    
    return &MyAdapter{
        BaseAdapter: NewBaseAdapter(capabilities, nil),
    }
}

func (a *MyAdapter) Transcribe(ctx context.Context, params TranscriptionParams) (*TranscriptionResult, error) {
    // Implementation
}
```

2. **Register Adapter**:
```go
func init() {
    registry.GetRegistry().Register(NewMyAdapter())
}
```

3. **Test**:
```bash
go test ./internal/transcription/adapters -run TestMyAdapter
```

### Adding a Frontend Component

1. **Create Component** (`web/frontend/src/components/MyComponent.tsx`):
```tsx
interface MyComponentProps {
  title: string;
  onAction: () => void;
}

export function MyComponent({ title, onAction }: MyComponentProps) {
  return (
    <div className="my-component">
      <h2>{title}</h2>
      <button onClick={onAction}>Action</button>
    </div>
  );
}
```

2. **Use Component**:
```tsx
import { MyComponent } from './components/MyComponent';

function App() {
  return <MyComponent title="Test" onAction={() => alert('Clicked')} />;
}
```

3. **Test**:
```tsx
test('MyComponent renders', () => {
  render(<MyComponent title="Test" onAction={jest.fn()} />);
  expect(screen.getByText('Test')).toBeInTheDocument();
});
```

## Database Migrations

### Auto-Migration

GORM auto-migrates on startup. Add new fields to models:

```go
type User struct {
    // ... existing fields
    NewField string `json:"new_field" gorm:"type:varchar(100)"`
}
```

Restart server - field automatically added.

### Manual Migration

For complex changes:

```go
func migrate() error {
    // Add column
    if err := database.DB.Exec("ALTER TABLE users ADD COLUMN new_field TEXT").Error; err != nil {
        return err
    }
    
    // Create index
    if err := database.DB.Exec("CREATE INDEX idx_users_new_field ON users(new_field)").Error; err != nil {
        return err
    }
    
    return nil
}
```

## Common Tasks

### Regenerate Swagger Docs

```bash
# Install swag
go install github.com/swaggo/swag/cmd/swag@latest

# Generate
swag init -g cmd/server/main.go -o api-docs
```

### Clear Database

```bash
rm data/scriberr.db
# Restart server to recreate
```

### Reset Python Environment

```bash
rm -rf whisperx-env nvidia-env
# Restart server - environments recreated on first transcription
```

### View Logs

```bash
# Set debug level
export LOG_LEVEL=debug

# Run with verbose output
go run cmd/server/main.go
```

## CI/CD

### GitHub Actions

**Workflow** (`.github/workflows/build.yml`):
```yaml
name: Build and Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.24'
      
      - name: Run tests
        run: go test ./...
      
      - name: Build
        run: go build -o scriberr cmd/server/main.go
```

## Docker Development

**Development Compose**:
```yaml
version: '3.9'
services:
  scriberr:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - ./data:/app/data
      - .:/app/src
    environment:
      - LOG_LEVEL=debug
```

**Build and run**:
```bash
docker-compose up --build
```

## Troubleshooting

### Port Already in Use

```bash
# Find process using port 8080
lsof -i :8080

# Kill process
kill -9 <PID>

# Or use different port
PORT=8081 go run cmd/server/main.go
```

### Go Module Issues

```bash
# Clean module cache
go clean -modcache

# Verify modules
go mod verify

# Tidy modules
go mod tidy
```

### Frontend Build Errors

```bash
# Clear node modules
rm -rf web/frontend/node_modules
cd web/frontend
npm install
```

### Database Locked

SQLite can lock with concurrent writes:
```bash
# Kill any running instances
killall scriberr

# Remove lock file
rm data/scriberr.db-wal data/scriberr.db-shm
```

## Contributing

### Pull Request Process

1. **Fork & Clone**:
```bash
git clone https://github.com/YOUR_USERNAME/Scriberr.git
cd Scriberr
```

2. **Create Branch**:
```bash
git checkout -b feature/my-feature
```

3. **Make Changes**:
- Follow coding standards
- Add tests
- Update documentation

4. **Commit**:
```bash
git add .
git commit -m "Add feature: description"
```

5. **Push & PR**:
```bash
git push origin feature/my-feature
# Create PR on GitHub
```

### Commit Messages

Follow conventional commits:
- `feat: add new feature`
- `fix: resolve bug`
- `docs: update documentation`
- `refactor: improve code structure`
- `test: add tests`
- `chore: update dependencies`

## Resources

- **Go Documentation**: https://go.dev/doc/
- **React Documentation**: https://react.dev/
- **GORM Documentation**: https://gorm.io/docs/
- **Vite Documentation**: https://vitejs.dev/
- **Tailwind CSS**: https://tailwindcss.com/docs

## Getting Help

- **GitHub Issues**: Report bugs, request features
- **GitHub Discussions**: Ask questions, share ideas
- **Documentation**: Check docs/ folder
- **Code**: Read existing code for examples

Happy coding! 🚀
