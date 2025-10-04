# Frontend Architecture

## Overview

The Scriberr frontend is a modern React application built with TypeScript, Vite, and Tailwind CSS. It provides an intuitive interface for managing audio transcriptions, chatting with transcripts, and configuring the system.

**Tech Stack**:
- **Framework**: React 18 with TypeScript
- **Build Tool**: Vite
- **Styling**: Tailwind CSS
- **UI Components**: shadcn/ui
- **Icons**: Lucide React
- **State Management**: React Context API
- **Routing**: React Router DOM

## Project Structure

```
web/frontend/
├── src/
│   ├── App.tsx                 # Main app component with routing
│   ├── main.tsx               # Application entry point
│   ├── index.css              # Global styles and Tailwind imports
│   │
│   ├── pages/                 # Page-level components
│   │   ├── Login.tsx          # Login page
│   │   ├── Register.tsx       # Registration page
│   │   ├── Settings.tsx       # Settings page
│   │   └── ChatPage.tsx       # Chat interface page
│   │
│   ├── components/            # Reusable components
│   │   ├── ui/               # shadcn/ui base components
│   │   ├── AudioRecorder.tsx
│   │   ├── TranscriptionConfigDialog.tsx
│   │   ├── ProfilesTable.tsx
│   │   ├── LLMSettings.tsx
│   │   ├── SummaryTemplatesTable.tsx
│   │   └── ...
│   │
│   ├── contexts/              # React Context providers
│   │   └── AuthContext.tsx   # Authentication context
│   │
│   ├── lib/                   # Utilities
│   │   └── utils.ts          # Helper functions
│   │
│   └── types/                 # TypeScript type definitions
│       └── note.ts
│
├── public/                    # Static assets
├── vite.config.ts            # Vite configuration
├── tsconfig.json             # TypeScript configuration
├── tailwind.config.ts        # Tailwind configuration
└── package.json              # Dependencies
```

## Core Components

### App.tsx

Main application component with routing:

```tsx
function App() {
  return (
    <AuthProvider>
      <Router>
        <Routes>
          <Route path="/login" element={<Login />} />
          <Route path="/register" element={<Register />} />
          <Route path="/" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
          <Route path="/transcription/:id" element={<ProtectedRoute><TranscriptView /></ProtectedRoute>} />
          <Route path="/chat/:id" element={<ProtectedRoute><ChatPage /></ProtectedRoute>} />
          <Route path="/settings" element={<ProtectedRoute><Settings /></ProtectedRoute>} />
        </Routes>
      </Router>
    </AuthProvider>
  );
}
```

### Authentication Context

**File**: `src/contexts/AuthContext.tsx`

Manages user authentication state and provides auth methods to all components:

```tsx
interface AuthContextType {
  user: User | null;
  login: (username: string, password: string) => Promise<void>;
  logout: () => void;
  register: (username: string, password: string) => Promise<void>;
  isLoading: boolean;
}

export function AuthProvider({ children }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // Check for existing session on mount
  useEffect(() => {
    const token = localStorage.getItem('access_token');
    if (token) {
      // Validate token and fetch user
      validateToken(token).then(setUser).finally(() => setIsLoading(false));
    } else {
      setIsLoading(false);
    }
  }, []);

  const login = async (username: string, password: string) => {
    const response = await fetch('/api/v1/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password })
    });
    
    const data = await response.json();
    localStorage.setItem('access_token', data.access_token);
    localStorage.setItem('refresh_token', data.refresh_token);
    setUser({ username });
  };

  const logout = () => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, register, isLoading }}>
      {children}
    </AuthContext.Provider>
  );
}
```

**Usage**:
```tsx
function SomeComponent() {
  const { user, logout } = useAuth();
  
  return (
    <div>
      <p>Welcome {user?.username}</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

## Key Features

### 1. Audio Upload & Transcription

**Component**: `TranscriptionConfigDialog.tsx`

Multi-step dialog for configuring and starting transcriptions:

```tsx
function TranscriptionConfigDialog() {
  const [file, setFile] = useState<File | null>(null);
  const [config, setConfig] = useState({
    model: 'medium',
    language: 'en',
    diarization: false,
    // ... more parameters
  });

  const handleUpload = async () => {
    // 1. Upload file
    const formData = new FormData();
    formData.append('file', file);
    const uploadRes = await fetch('/api/v1/transcription/upload', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}` },
      body: formData
    });
    const { id } = await uploadRes.json();

    // 2. Start transcription with config
    await fetch(`/api/v1/transcription/${id}/start`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(config)
    });

    // 3. Navigate to job page
    navigate(`/transcription/${id}`);
  };

  return (
    <Dialog>
      {/* File selection */}
      {/* Model configuration */}
      {/* Diarization settings */}
      <Button onClick={handleUpload}>Start Transcription</Button>
    </Dialog>
  );
}
```

### 2. Transcript Viewer

**Features**:
- Audio player with playback controls
- Scrollable transcript with timestamps
- Click word to seek in audio
- Highlight current segment during playback
- Speaker labels (if diarization enabled)
- Add notes at specific timestamps

**Implementation**:

```tsx
function TranscriptView() {
  const { id } = useParams();
  const [transcript, setTranscript] = useState(null);
  const [currentTime, setCurrentTime] = useState(0);
  const audioRef = useRef<HTMLAudioElement>(null);

  // Fetch transcript
  useEffect(() => {
    fetch(`/api/v1/transcription/${id}/transcript`, {
      headers: { 'Authorization': `Bearer ${token}` }
    })
      .then(res => res.json())
      .then(setTranscript);
  }, [id]);

  // Update current time as audio plays
  useEffect(() => {
    const audio = audioRef.current;
    const handleTimeUpdate = () => setCurrentTime(audio.currentTime);
    audio?.addEventListener('timeupdate', handleTimeUpdate);
    return () => audio?.removeEventListener('timeupdate', handleTimeUpdate);
  }, []);

  // Seek to timestamp when clicking word
  const handleWordClick = (timestamp: number) => {
    if (audioRef.current) {
      audioRef.current.currentTime = timestamp;
      audioRef.current.play();
    }
  };

  // Determine which segment is currently playing
  const activeSegment = transcript?.segments.find(
    seg => currentTime >= seg.start && currentTime <= seg.end
  );

  return (
    <div>
      <audio ref={audioRef} src={`/api/v1/transcription/${id}/audio`} />
      <AudioControls audioRef={audioRef} />
      
      <div className="transcript">
        {transcript?.segments.map(segment => (
          <div 
            key={segment.start}
            className={segment === activeSegment ? 'active' : ''}
          >
            <span className="speaker">{segment.speaker}</span>
            <span className="time">{formatTime(segment.start)}</span>
            <span className="text">
              {segment.words.map(word => (
                <span 
                  key={word.start}
                  onClick={() => handleWordClick(word.start)}
                  className="word"
                >
                  {word.word}
                </span>
              ))}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 3. Chat Interface

**File**: `src/pages/ChatPage.tsx`

LLM-powered chat over transcripts with streaming responses:

```tsx
function ChatPage() {
  const { id } = useParams(); // transcription ID
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);

  // Load existing chat session
  useEffect(() => {
    fetch(`/api/v1/chat/transcriptions/${id}/sessions`, {
      headers: { 'Authorization': `Bearer ${token}` }
    })
      .then(res => res.json())
      .then(sessions => {
        if (sessions.length > 0) {
          loadSession(sessions[0].id);
        }
      });
  }, [id]);

  const sendMessage = async () => {
    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsStreaming(true);

    // Stream response
    const response = await fetch(`/api/v1/chat/sessions/${sessionId}/messages`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        content: input,
        model: 'gpt-4',
        temperature: 0.7
      })
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let assistantMessage = { role: 'assistant', content: '' };
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = JSON.parse(line.slice(6));
          if (data.content) {
            assistantMessage.content += data.content;
            setMessages(prev => [...prev.slice(0, -1), { ...assistantMessage }]);
          }
        }
      }
    }

    setIsStreaming(false);
  };

  return (
    <div className="chat-container">
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={`message ${msg.role}`}>
            {msg.content}
          </div>
        ))}
      </div>
      
      <div className="input-area">
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          onKeyPress={e => e.key === 'Enter' && sendMessage()}
          placeholder="Ask about the transcript..."
          disabled={isStreaming}
        />
        <button onClick={sendMessage} disabled={isStreaming}>
          Send
        </button>
      </div>
    </div>
  );
}
```

### 4. Settings Page

**File**: `src/pages/Settings.tsx`

Tabbed interface for:
- **LLM Configuration**: OpenAI API key or Ollama URL
- **Profiles**: Manage transcription profiles
- **API Keys**: Generate and manage API keys
- **Summary Templates**: Create reusable summary prompts

```tsx
function Settings() {
  const [activeTab, setActiveTab] = useState('llm');

  return (
    <div className="settings">
      <Tabs value={activeTab} onValueChange={setActiveTab}>
        <TabsList>
          <TabsTrigger value="llm">LLM Settings</TabsTrigger>
          <TabsTrigger value="profiles">Profiles</TabsTrigger>
          <TabsTrigger value="api-keys">API Keys</TabsTrigger>
          <TabsTrigger value="templates">Templates</TabsTrigger>
        </TabsList>

        <TabsContent value="llm">
          <LLMSettings />
        </TabsContent>

        <TabsContent value="profiles">
          <ProfilesTable />
        </TabsContent>

        <TabsContent value="api-keys">
          <APIKeysTable />
        </TabsContent>

        <TabsContent value="templates">
          <SummaryTemplatesTable />
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

### 5. Audio Recorder

**Component**: `AudioRecorder.tsx`

Record audio directly in the browser:

```tsx
function AudioRecorder() {
  const [isRecording, setIsRecording] = useState(false);
  const [audioBlob, setAudioBlob] = useState<Blob | null>(null);
  const mediaRecorderRef = useRef<MediaRecorder | null>(null);

  const startRecording = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    const mediaRecorder = new MediaRecorder(stream);
    mediaRecorderRef.current = mediaRecorder;

    const chunks: Blob[] = [];
    mediaRecorder.ondataavailable = e => chunks.push(e.data);
    mediaRecorder.onstop = () => {
      const blob = new Blob(chunks, { type: 'audio/webm' });
      setAudioBlob(blob);
    };

    mediaRecorder.start();
    setIsRecording(true);
  };

  const stopRecording = () => {
    mediaRecorderRef.current?.stop();
    setIsRecording(false);
  };

  const uploadRecording = async () => {
    const formData = new FormData();
    formData.append('file', audioBlob, 'recording.webm');
    
    await fetch('/api/v1/transcription/upload', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}` },
      body: formData
    });
  };

  return (
    <div>
      {!isRecording ? (
        <button onClick={startRecording}>Start Recording</button>
      ) : (
        <button onClick={stopRecording}>Stop Recording</button>
      )}
      
      {audioBlob && (
        <button onClick={uploadRecording}>Upload & Transcribe</button>
      )}
    </div>
  );
}
```

## UI Components (shadcn/ui)

Scriberr uses [shadcn/ui](https://ui.shadcn.com/) for base components:

**Installed Components**:
- Button
- Input
- Dialog
- Tabs
- Table
- Card
- Select
- Textarea
- Toast
- DropdownMenu
- Progress

**Example Usage**:
```tsx
import { Button } from '@/components/ui/button';
import { Dialog, DialogContent, DialogTitle } from '@/components/ui/dialog';

function MyComponent() {
  return (
    <Dialog>
      <DialogContent>
        <DialogTitle>Upload Audio</DialogTitle>
        <Button>Submit</Button>
      </DialogContent>
    </Dialog>
  );
}
```

## Styling

### Tailwind CSS

**Configuration**: `tailwind.config.ts`

```ts
export default {
  darkMode: 'class',
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: {...},
        secondary: {...}
      }
    }
  }
}
```

**Usage**:
```tsx
<div className="flex items-center justify-between p-4 bg-gray-100 rounded-lg">
  <h2 className="text-xl font-bold text-gray-900">Title</h2>
  <button className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
    Click Me
  </button>
</div>
```

## API Communication

### Fetch Wrapper

Common pattern for API calls:

```tsx
async function apiCall(endpoint: string, options = {}) {
  const token = localStorage.getItem('access_token');
  
  const response = await fetch(`/api/v1${endpoint}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      ...options.headers
    }
  });

  if (response.status === 401) {
    // Token expired, try refresh
    await refreshToken();
    return apiCall(endpoint, options);
  }

  if (!response.ok) {
    throw new Error(await response.text());
  }

  return response.json();
}
```

### Token Refresh

```tsx
async function refreshToken() {
  const refreshToken = localStorage.getItem('refresh_token');
  
  const response = await fetch('/api/v1/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refresh_token: refreshToken })
  });

  const data = await response.json();
  localStorage.setItem('access_token', data.access_token);
  localStorage.setItem('refresh_token', data.refresh_token);
}
```

## State Management

### React Context API

Used for global state:
- **AuthContext**: User authentication
- **ThemeContext**: Dark/light mode (if implemented)

### Component State

Local state with `useState`:
```tsx
const [jobs, setJobs] = useState<Job[]>([]);
const [isLoading, setIsLoading] = useState(false);
```

### Data Fetching

Using `useEffect`:
```tsx
useEffect(() => {
  setIsLoading(true);
  
  fetch('/api/v1/transcription/list')
    .then(res => res.json())
    .then(setJobs)
    .finally(() => setIsLoading(false));
}, []);
```

## Build Process

### Development

```bash
cd web/frontend
npm run dev
```

**Vite Dev Server**:
- Hot Module Replacement (HMR)
- Fast refresh
- Source maps
- Proxy to backend (if needed)

### Production Build

```bash
npm run build
```

**Output**: `dist/`
- Minified JavaScript
- Optimized CSS
- Tree-shaken code
- Asset hashing

### Embedding in Go

```bash
# From repo root
./build.sh
```

1. Builds React app → `web/frontend/dist`
2. Copies to `internal/web/dist`
3. Go embeds files using `embed` package
4. Compiles Go binary with embedded frontend

## Routing

### React Router

**Routes**:
- `/` - Dashboard (job list)
- `/login` - Login page
- `/register` - Registration
- `/transcription/:id` - View transcript
- `/chat/:id` - Chat with transcript
- `/settings` - Settings page

**Protected Routes**:
```tsx
function ProtectedRoute({ children }) {
  const { user, isLoading } = useAuth();

  if (isLoading) return <Loading />;
  if (!user) return <Navigate to="/login" />;
  
  return children;
}
```

## Type Safety

### TypeScript Interfaces

```tsx
interface TranscriptionJob {
  id: string;
  title?: string;
  status: 'uploaded' | 'pending' | 'processing' | 'completed' | 'failed';
  audio_path: string;
  created_at: string;
  updated_at: string;
  parameters: WhisperXParams;
}

interface ChatMessage {
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: string;
}
```

**Usage**:
```tsx
const [job, setJob] = useState<TranscriptionJob | null>(null);
```

## Performance Optimizations

### Code Splitting

```tsx
const ChatPage = lazy(() => import('./pages/ChatPage'));

<Suspense fallback={<Loading />}>
  <ChatPage />
</Suspense>
```

### Memoization

```tsx
const expensiveValue = useMemo(() => {
  return computeExpensiveValue(data);
}, [data]);
```

### Debouncing

```tsx
const debouncedSearch = useMemo(
  () => debounce((query) => searchTranscripts(query), 300),
  []
);
```

## Testing

### Unit Tests

```tsx
import { render, screen } from '@testing-library/react';
import { Button } from './Button';

test('renders button with text', () => {
  render(<Button>Click me</Button>);
  expect(screen.getByText('Click me')).toBeInTheDocument();
});
```

## Best Practices

1. **Component Organization**: One component per file
2. **Type Everything**: Use TypeScript interfaces
3. **Extract Hooks**: Reusable logic in custom hooks
4. **Error Boundaries**: Catch component errors
5. **Loading States**: Show loading indicators
6. **Error Handling**: Display user-friendly errors
7. **Accessibility**: ARIA labels, keyboard navigation
8. **Responsive Design**: Mobile-friendly layouts
9. **Code Splitting**: Lazy load heavy components
10. **Clean Up**: Cancel requests, remove listeners

## Future Enhancements

- Dark mode support
- Offline mode (service worker)
- Real-time updates (WebSocket)
- Drag-and-drop file upload
- Keyboard shortcuts
- Export transcript as PDF
- Waveform visualization
- Multi-language UI
