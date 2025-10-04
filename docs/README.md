# Scriberr Documentation

Welcome to the comprehensive Scriberr documentation! This folder contains detailed guides for understanding, using, developing, and deploying Scriberr.

## Documentation Index

### 📖 [OVERVIEW.md](OVERVIEW.md)
High-level introduction to Scriberr covering features, architecture, and target users. **Start here** if you're new to Scriberr.

**Topics**:
- What is Scriberr?
- Core features
- Technology stack
- Target users
- System requirements
- Getting started

### 🏗️ [ARCHITECTURE.md](ARCHITECTURE.md)
Deep dive into Scriberr's system architecture, component design, and technical implementation.

**Topics**:
- Three-tier architecture
- Frontend (React)
- Backend (Go)
- Processing layer (Python)
- Database schema
- Job queue system
- Data flow
- Security architecture
- Performance optimizations

### 🔌 [API.md](API.md)
Complete REST API reference with authentication, endpoints, and examples.

**Topics**:
- Authentication (JWT & API keys)
- Transcription endpoints
- Chat endpoints
- Profiles and templates
- Error handling
- Code examples (cURL, Python, JavaScript)

### 🎙️ [TRANSCRIPTION.md](TRANSCRIPTION.md)
Detailed guide to the transcription engine, models, and processing.

**Topics**:
- Adapter architecture
- Supported models (WhisperX, Pyannote, NVIDIA)
- Model selection guide
- Python environment management
- Transcription workflow
- Output formats
- Performance optimization
- Troubleshooting

### 💾 [DATABASE.md](DATABASE.md)
Database schema, models, and data management.

**Topics**:
- SQLite + GORM setup
- Core models (User, TranscriptionJob, Note, etc.)
- Relationships
- Queries and indexes
- Migrations
- Backup and restore
- Best practices

### 🎨 [FRONTEND.md](FRONTEND.md)
Frontend architecture, components, and development.

**Topics**:
- React + TypeScript setup
- Component structure
- State management
- Audio player implementation
- Chat interface
- API communication
- UI components (shadcn/ui)
- Build process

### 🚀 [DEPLOYMENT.md](DEPLOYMENT.md)
Production deployment options and configurations.

**Topics**:
- Homebrew installation
- Docker deployment
- Docker Compose
- GPU support (NVIDIA)
- Binary downloads
- Build from source
- Reverse proxy (Nginx, Apache, Caddy)
- SSL/TLS setup
- Cloud deployment (AWS, GCP, DigitalOcean)
- Kubernetes
- Monitoring and backups

### 💻 [DEVELOPMENT.md](DEVELOPMENT.md)
Local development setup and contribution guide.

**Topics**:
- Prerequisites and setup
- Development workflow
- Project structure
- Coding standards (Go, TypeScript)
- Testing
- Debugging
- Adding new features
- Database migrations
- CI/CD
- Troubleshooting

### 🔄 [WORKFLOW.md](WORKFLOW.md)
End-to-end workflows for common use cases.

**Topics**:
- Basic transcription
- Multi-speaker diarization
- YouTube transcription
- Chat with transcripts
- Automated summarization
- Note-taking
- Reusable profiles
- Batch processing
- GPU acceleration
- API integration

## Quick Links

### For New Users
1. [OVERVIEW.md](OVERVIEW.md) - Understand what Scriberr is
2. [DEPLOYMENT.md](DEPLOYMENT.md) - Install and run Scriberr
3. [WORKFLOW.md](WORKFLOW.md) - Follow step-by-step guides

### For Developers
1. [ARCHITECTURE.md](ARCHITECTURE.md) - Understand the system design
2. [DEVELOPMENT.md](DEVELOPMENT.md) - Set up development environment
3. [API.md](API.md) - API reference for integrations

### For Power Users
1. [TRANSCRIPTION.md](TRANSCRIPTION.md) - Master transcription options
2. [API.md](API.md) - Automate with API
3. [WORKFLOW.md](WORKFLOW.md) - Advanced workflows

## Additional Resources

### Official Documentation
- **Website**: https://scriberr.app
- **Installation Guide**: https://scriberr.app/docs/installation.html
- **API Reference**: https://scriberr.app/api.html
- **Changelog**: https://scriberr.app/changelog.html

### Community
- **GitHub Repository**: https://github.com/rishikanthc/Scriberr
- **Issues**: https://github.com/rishikanthc/Scriberr/issues
- **Discussions**: https://github.com/rishikanthc/Scriberr/discussions

### API Documentation
- **Swagger UI**: `/swagger/index.html` on your Scriberr instance
- **OpenAPI Spec**: `/api/swagger.json`

## Documentation Structure

```
docs/
├── README.md              # This file
├── OVERVIEW.md           # Introduction and features
├── ARCHITECTURE.md       # System architecture
├── API.md                # REST API reference
├── TRANSCRIPTION.md      # Transcription engine guide
├── DATABASE.md           # Database schema and models
├── FRONTEND.md           # Frontend architecture
├── DEPLOYMENT.md         # Deployment guide
├── DEVELOPMENT.md        # Development guide
└── WORKFLOW.md           # End-to-end workflows
```

## Contributing to Documentation

Found an error or want to improve the docs?

1. **Fork the repository**
2. **Edit documentation files** (Markdown)
3. **Submit a pull request**

Documentation guidelines:
- Use clear, simple language
- Include code examples
- Add screenshots where helpful
- Keep formatting consistent
- Update the index if adding new files

## License

This documentation is part of Scriberr and is licensed under the MIT License.

---

**Need help?** Open a [GitHub Issue](https://github.com/rishikanthc/Scriberr/issues) or start a [Discussion](https://github.com/rishikanthc/Scriberr/discussions).
