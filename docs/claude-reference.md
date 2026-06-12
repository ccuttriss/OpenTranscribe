# Claude Code Reference — OpenTranscribe

Deep-dive reference moved out of CLAUDE.md (2026-06-11) to keep session context lean.
CLAUDE.md holds the rules and daily commands; this file holds the detail.

## Multi-GPU Worker Scaling (Optional)

For systems with multiple GPUs, you can enable parallel GPU workers to significantly increase transcription throughput.

**Use Case**: You have multiple GPUs and want to maximize processing speed by running multiple transcription workers in parallel.

**Example Hardware Setup**:
- GPU 0: NVIDIA RTX A6000 (49GB) - Running LLM model
- GPU 1: RTX 3080 Ti (12GB) - Default single worker (disabled when scaling)
- GPU 2: NVIDIA RTX A6000 (49GB) - Scaled workers (4 parallel)

**Configuration** (in `.env`):
```bash
GPU_SCALE_ENABLED=true      # Enable multi-GPU scaling
GPU_SCALE_DEVICE_ID=2       # Which GPU to use (default: 2)
GPU_SCALE_WORKERS=4         # Number of parallel workers (default: 4)
```

**Usage**:
```bash
# Start with GPU scaling enabled
./opentr.sh start dev --gpu-scale

# Reset with GPU scaling enabled
./opentr.sh reset dev --gpu-scale

# View scaled worker logs
docker compose logs -f celery-worker-gpu-scaled
```

**How It Works**:
- When `--gpu-scale` flag is used, the system loads `docker-compose.gpu-scale.yml` overlay
- Default single GPU worker is disabled (`scale: 0`)
- A new single container is created with `concurrency=4` (configurable via `GPU_SCALE_WORKERS`)
- The container runs 4 parallel Celery workers within a single process
- All workers target the specified GPU device and process from the `gpu` queue
- Celery automatically distributes tasks across the worker pool

**Performance**: With 4 parallel workers on a high-end GPU like the A6000, you can process 4 videos simultaneously, significantly reducing total processing time for batches of media files.

**Scaling**: Simply change `GPU_SCALE_WORKERS` in your `.env` file to adjust the number of concurrent workers (e.g., 2, 4, 6, 8) based on your GPU's memory and processing capacity.

### Docker Build & Push (Production Images)

Build and push production Docker images to Docker Hub:

```bash
# Build and push both services
./scripts/docker-build-push.sh

# Build specific service only
./scripts/docker-build-push.sh backend
./scripts/docker-build-push.sh frontend

# Auto-detect changes and build only what changed
./scripts/docker-build-push.sh auto

# Build for single platform (faster testing)
PLATFORMS=linux/amd64 ./scripts/docker-build-push.sh backend
```

See [scripts/README.md](scripts/README.md) for detailed documentation.

### Local Development Builds (No Docker Hub Push)

When making code changes and testing locally without pushing to Docker Hub:

```bash
# Build backend image locally (always use --no-cache for code changes)
cd ~/prj/src/OpenTranscribe
docker build --no-cache -t davidamacey/opentranscribe-backend:latest -f backend/Dockerfile.prod backend/

# Restart production with local image (use docker-compose.local.yml to prevent pulling)
cd ~/prj/opentranscribe
docker compose -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.local.yml up -d --force-recreate
```

**Important:**
- `--no-cache` is required because Docker may cache the COPY layer even when files changed
- `--force-recreate` ensures containers use the new image
- `docker-compose.local.yml` sets `pull_policy: never` to prevent overwriting local images with Docker Hub versions
- Never use `./opentranscribe.sh update` with local builds (it pulls from Docker Hub)

## Authentication System (deep dive)

OpenTranscribe supports multiple authentication methods that can be enabled simultaneously (hybrid authentication).

**Available Authentication Methods:**
- **Local (Direct)**: Username/password with bcrypt hashing
- **LDAP/Active Directory**: Enterprise directory integration
- **OIDC/Keycloak**: OpenID Connect with external identity providers
- **PKI/X.509**: Certificate-based authentication for high-security environments

**Authentication Module Structure:**
```
backend/app/auth/
├── direct_auth.py       # Local password authentication
├── ldap_auth.py         # LDAP/Active Directory integration
├── keycloak_auth.py     # OIDC/Keycloak integration
├── pki_auth.py          # PKI/X.509 certificate authentication
├── mfa.py               # TOTP multi-factor authentication
├── password_policy.py   # Password strength enforcement
├── password_history.py  # Password reuse prevention
├── rate_limit.py        # Authentication rate limiting
├── lockout.py           # Account lockout management
├── session.py           # Session/token management
├── token_service.py     # JWT token operations
├── audit.py             # Authentication audit logging
└── constants.py         # Auth-related constants
```

**Authentication Database Models:**
- `UserMFA` - MFA/TOTP settings per user (`backend/app/models/user_mfa.py`)
- `PasswordHistory` - Password history tracking (`backend/app/models/password_history.py`)
- `RefreshToken` - Refresh token management (`backend/app/models/refresh_token.py`)

**Key Authentication Patterns:**
- **Hybrid Auth**: Multiple methods can be enabled simultaneously via `AUTH_TYPE` config
- **Token Flow**: JWT access tokens (short-lived) + refresh token rotation (long-lived)
- **External IdP Users**: PKI/Keycloak users bypass local MFA (handled by IdP)
- **Rate Limiting**: Per-IP and per-user rate limits to prevent brute force
- **Account Lockout**: Configurable lockout after failed attempts
- **Audit Logging**: All auth events logged for security compliance

**Configuration:**
- Set `AUTH_TYPE` in `.env` (local, ldap, keycloak, pki, or comma-separated for hybrid)
- Provider-specific settings documented in `docs/KEYCLOAK_SETUP.md` and `docs/PKI_SETUP.md`
- MFA can be enforced globally or per-user

## AI Processing Workflow

1. File upload to MinIO storage
2. Metadata extraction and database record creation
3. Celery task dispatch to worker with GPU support
4. WhisperX transcription with word-level alignment (100+ languages supported)
   - Configurable source language (auto-detect or specify)
   - Optional translation to English
   - ~42 languages support word-level timestamps
5. PyAnnote speaker diarization and voice fingerprinting
6. **LLM speaker identification suggestions** (optional - manual verification required)
7. **LLM-powered summarization** with BLUF format (optional - user-triggered)
   - Automatic context-aware processing (single or multi-section based on transcript length)
   - Intelligent chunking at speaker/topic boundaries for long content
   - Section-by-section analysis with final summary stitching
   - Configurable output language (12 languages supported)
8. Database storage and OpenSearch indexing
9. WebSocket notification to frontend

### LLM Features

The application now includes optional AI-powered features using Large Language Models:

**AI Summarization:**
- BLUF (Bottom Line Up Front) format summaries
- Speaker analysis with talk time and key contributions
- Action items extraction with priorities and assignments
- Key decisions and follow-up items identification
- Support for multiple LLM providers (vLLM, OpenAI, Ollama, Claude)
- Intelligent section-by-section processing for transcripts of any length
- Automatic context-aware chunking and summary stitching
- **Multilingual output**: Generate summaries in 12 languages (en, es, fr, de, pt, zh, ja, ko, it, ru, ar, hi)

**Speaker Identification:**
- LLM-powered speaker name suggestions based on conversation context
- Confidence scoring for identification accuracy
- Manual verification workflow (suggestions are not auto-applied)
- Cross-video speaker matching with embedding analysis
- Speaker merge UI for combining duplicate speakers

**Model Discovery:**
- Automatic discovery of available models for vLLM, Ollama, and Anthropic
- Works with OpenAI-compatible API endpoints
- Edit mode supports stored API keys (no need to re-enter)

**Configuration:**
- Set `LLM_PROVIDER` in .env file (vllm, openai, ollama, anthropic, openrouter)
- Configure provider-specific settings (API keys, endpoints, models)
- Features work independently - transcription works without LLM configuration

**Deployment Options:**
- **Cloud Providers**: Use `.env` configuration with external providers (OpenAI, Claude, OpenRouter, etc.)
- **Self-Hosted LLM**: Configure vLLM or Ollama endpoints in `.env` (deployed separately)
- **No LLM**: Leave LLM_PROVIDER empty for transcription-only mode

### Multilingual Transcription (New)

OpenTranscribe now supports transcription in 100+ languages with configurable settings:

**User-Configurable Language Settings:**
- **Source Language**: Auto-detect or specify language for improved accuracy
- **Translate to English**: Toggle to translate non-English audio to English
- **LLM Output Language**: Generate AI summaries in preferred language (12 supported)

**Language Features:**
- ~42 languages support word-level timestamps via wav2vec2 alignment models
- Languages without alignment models fall back to segment-level timestamps
- Settings stored per-user in the database

**Configuration:**
- User settings available in Settings → Transcription → Language Settings
- Per-file language override available at upload/reprocess time

### User-Level Settings

Users can configure their own transcription preferences:

**Transcription Settings:**
- **Speaker Behavior**: Always prompt, use defaults, or use saved custom values
- **Min/Max Speakers**: Configure default speaker detection range (1-50+)
- **Garbage Cleanup**: Enable/disable automatic cleanup of erroneous segments

**Recording Settings:**
- Audio recording quality and duration preferences
- Microphone device selection

### Universal Media URL Support

OpenTranscribe supports downloading and processing videos from 1800+ platforms via yt-dlp integration:

**Supported Platforms:**
- **Best Support (Recommended)**: YouTube, Dailymotion, TikTok - most reliable for public video downloads
- **Limited Support**: Vimeo, Twitter/X, Instagram, Facebook - may require authentication for many videos
- **Other Platforms**: Twitch, Reddit, SoundCloud, and 1800+ more sites supported by yt-dlp

**Platform Limitations:**
- Some platforms (Vimeo, Instagram, Facebook) restrict most videos to logged-in users
- Authentication-required videos cannot be downloaded without browser cookies
- Age-restricted, geo-restricted, or private videos are not accessible
- Premium/subscriber-only content requires active subscriptions

**User-Friendly Error Messages:**
- The system detects authentication-related errors and provides helpful guidance
- When a video fails, users receive platform-specific suggestions
- Recommends alternative platforms when authentication issues are detected

**How It Works:**
1. User enters a video URL from any supported platform
2. System validates the URL and extracts video metadata via yt-dlp
3. Video is downloaded in web-compatible format (H.264/MP4 preferred)
4. Downloaded video is uploaded to MinIO storage
5. Standard transcription pipeline processes the video
6. WebSocket notification updates the frontend

**Configuration:**
- No additional configuration required - yt-dlp is included in the backend container
- Anti-blocking measures included for YouTube (client rotation, proper headers)
- Maximum video duration: 4 hours
- Maximum file size: 15GB (same as direct upload limit)

## Model Caching System

OpenTranscribe uses a simple volume-based model caching system that automatically persists AI models between container restarts.

### Configuration
- Set `MODEL_CACHE_DIR` in `.env` to specify cache location (default: `./models`)
- Models are automatically downloaded on first use
- All models persist across container restarts and rebuilds

### Directory Structure
```
${MODEL_CACHE_DIR}/
├── huggingface/          # HuggingFace models cache
│   ├── hub/             # WhisperX models (~1.5GB)
│   └── transformers/    # PyAnnote transformer cache
├── torch/               # PyTorch models cache
│   ├── hub/checkpoints/ # Wav2Vec2 alignment model (~360MB)
│   └── pyannote/        # PyAnnote speaker models (~500MB)
├── nltk_data/           # NLTK data files
│   ├── tokenizers/      # punkt_tab tokenizer (~13MB)
│   └── taggers/         # POS taggers
└── sentence-transformers/ # Sentence transformers models
    └── sentence-transformers_all-MiniLM-L6-v2/ # Semantic search model (~80MB)
```

### Speaker Diarization Configuration

**MIN_SPEAKERS / MAX_SPEAKERS Parameters:**

PyAnnote's speaker diarization uses sklearn's `AgglomerativeClustering`, which has **NO hard maximum limit** on the number of speakers:
- Default: `MIN_SPEAKERS=1`, `MAX_SPEAKERS=20`
- Can be increased to 50+ for large conferences/events with many speakers
- No hard upper limit - only constrained by the number of audio samples
- Performance threshold at `max(100, 0.02 * n_samples)` where algorithm behavior changes for efficiency

**Use Cases:**
- Small meetings: 2-5 speakers (default works fine)
- Medium meetings: 5-15 speakers (default works fine)
- Large conferences: 15-50 speakers (increase MAX_SPEAKERS to 30-50)
- Very large events: 50+ speakers (increase MAX_SPEAKERS accordingly)

**Note**: Higher values may impact processing time but will not cause errors.

### Docker Volume Mappings
The system uses simple volume mappings to cache models to their natural locations:
```yaml
volumes:
  - ${MODEL_CACHE_DIR}/huggingface:/home/appuser/.cache/huggingface
  - ${MODEL_CACHE_DIR}/torch:/home/appuser/.cache/torch
  - ${MODEL_CACHE_DIR}/nltk_data:/home/appuser/.cache/nltk_data
  - ${MODEL_CACHE_DIR}/sentence-transformers:/home/appuser/.cache/sentence-transformers
```

### Key Benefits
- **No code complexity**: Models use their natural cache locations
- **Persistent storage**: Models saved between container restarts
- **User configurable**: Simple `.env` variable controls cache location
- **No re-downloads**: Models cached after first download (~2.6GB total)

## Security: Non-Root Container User (detail)

OpenTranscribe backend containers run as a non-root user (`appuser`, UID 1000) following Docker security best practices.

**Benefits:**
- Follows principle of least privilege
- Reduces security risk from container escape vulnerabilities
- Compliant with security scanning tools (Trivy, Snyk, etc.)
- Prevents host root compromise in case of container breach

**Automatic Permission Management:**

The startup scripts (`./opentr.sh` and `./opentranscribe.sh`) automatically check and fix model cache permissions before starting containers. This ensures the non-root container user (UID 1000) can access the model cache without permission errors.

If you encounter permission issues, you can manually fix them:

```bash
# Fix permissions on existing model cache
./scripts/fix-model-permissions.sh
```

This script will change ownership of your model cache to UID:GID 1000:1000, making it accessible to the non-root container user.

**Technical Details:**
- Container user: `appuser` (UID 1000, GID 1000)
- User groups: `appuser`, `video` (for GPU access)
- Cache directories: `/home/appuser/.cache/huggingface`, `/home/appuser/.cache/torch`
- Multi-stage build for minimal attack surface
- Health checks for container orchestration

