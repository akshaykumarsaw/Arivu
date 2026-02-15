# Design Document: Arivu Medical AI - Education Edition (Phase 1)

## Overview

Arivu Medical AI - Education Edition is a full-stack web application that provides interactive learning tools for medical students and faculty. The platform leverages Google Gemini AI models for natural language understanding and generation, combined with custom visual generation engines for mind maps and infographics.

The architecture follows a modern microservices-inspired approach with clear separation between frontend (Next.js 14), backend API (NestJS), AI orchestration layer, and vector storage (Qdrant) for document retrieval.

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend Layer                          │
│  Next.js 14 + React + Tailwind CSS + Radix UI              │
│  - AI Tutor Interface                                       │
│  - Quiz Generator UI                                        │
│  - Mind Map Viewer (React Flow)                            │
│  - Infographic Display                                      │
│  - Slide Creator Interface                                  │
│  - Document Upload & Chat                                   │
└────────────────────┬────────────────────────────────────────┘
                     │ REST API / WebSocket
┌────────────────────┴────────────────────────────────────────┐
│                     Backend API Layer                       │
│  NestJS + TypeScript                                        │
│  - Authentication Module (JWT)                              │
│  - AI Tutor Controller                                      │
│  - Quiz Generator Controller                                │
│  - Mind Map Controller                                      │
│  - Infographic Controller                                   │
│  - Slide Creator Controller                                 │
│  - Document Processing Controller                           │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────────┐
│                  AI Orchestration Layer                     │
│  - Gemini Service (Pro/Flash models)                        │
│  - Guard Agent Service                                      │
│  - Prompt Engineering Module                                │
│  - Context Management                                       │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────────┐
│                   Data & Storage Layer                      │
│  - PostgreSQL (User data, sessions, history)                │
│  - Qdrant Vector DB (Document embeddings)                   │
│  - Redis (Session cache, rate limiting)                     │
│  - S3-compatible storage (Uploaded documents, exports)      │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack Rationale

- **Next.js 14**: Server-side rendering for SEO, App Router for modern routing, React Server Components for performance
- **NestJS**: TypeScript-first framework with dependency injection, modular architecture, built-in validation
- **Google Gemini**: State-of-the-art multimodal AI with strong medical knowledge, cost-effective, fast inference
- **Qdrant**: High-performance vector database optimized for semantic search and RAG applications
- **PostgreSQL**: Reliable relational database for structured user and session data
- **Redis**: In-memory cache for session management and rate limiting
- **Radix UI**: Accessible, unstyled components for building custom UI with Tailwind CSS

## Components and Interfaces

### Frontend Components

#### 1. AI Tutor Chat Component

```typescript
interface ChatMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  validated: boolean;
}

interface ChatSession {
  id: string;
  userId: string;
  messages: ChatMessage[];
  context: string[];
  createdAt: Date;
  updatedAt: Date;
}

// Component Props
interface AITutorChatProps {
  sessionId?: string;
  onMessageSent: (message: string) => Promise<void>;
  onSessionCreated: (sessionId: string) => void;
  focusMode: boolean;
}
```

#### 2. Quiz Generator Component

```typescript
interface QuizQuestion {
  id: string;
  question: string;
  options: string[];
  correctAnswer: number;
  explanation: string;
  difficulty: 'recall' | 'application' | 'analysis';
  topic: string;
}

interface QuizSession {
  id: string;
  questions: QuizQuestion[];
  currentIndex: number;
  answers: Map<string, number>;
  score: number;
  completed: boolean;
}

// Component Props
interface QuizGeneratorProps {
  topic?: string;
  documentId?: string;
  questionCount: number;
  onQuizGenerated: (session: QuizSession) => void;
  onAnswerSubmitted: (questionId: string, answer: number) => void;
}
```

#### 3. Mind Map Viewer Component

```typescript
interface MindMapNode {
  id: string;
  label: string;
  type: 'symptom' | 'cause' | 'diagnosis' | 'treatment' | 'concept';
  description: string;
  expanded: boolean;
  children: string[];
}

interface MindMapEdge {
  id: string;
  source: string;
  target: string;
  label: string;
  type: 'causes' | 'treats' | 'indicates' | 'related';
}

interface MindMap {
  id: string;
  topic: string;
  nodes: MindMapNode[];
  edges: MindMapEdge[];
}

// Component Props
interface MindMapViewerProps {
  mindMap: MindMap;
  onNodeClick: (nodeId: string) => void;
  onNodeExpand: (nodeId: string) => void;
  onNodeCollapse: (nodeId: string) => void;
  onExport: (format: 'png' | 'svg') => Promise<Blob>;
}
```

#### 4. Infographic Generator Component

```typescript
type InfographicType = 'statistics' | 'flowchart' | 'risk-matrix' | 'comparison';

interface InfographicData {
  type: InfographicType;
  title: string;
  data: Record<string, any>;
  colorScheme: string[];
}

interface Infographic {
  id: string;
  type: InfographicType;
  svgContent: string;
  metadata: InfographicData;
}

// Component Props
interface InfographicGeneratorProps {
  topic: string;
  type: InfographicType;
  onGenerated: (infographic: Infographic) => void;
  onExport: (format: 'png' | 'svg') => Promise<Blob>;
}
```

#### 5. Slide Creator Component

```typescript
interface Slide {
  id: string;
  type: 'title' | 'content' | 'diagram' | 'references';
  title: string;
  content: string[];
  layout: 'bullets' | 'two-column' | 'diagram' | 'table';
  notes: string;
}

interface Presentation {
  id: string;
  topic: string;
  slides: Slide[];
  theme: string;
  metadata: {
    author: string;
    createdAt: Date;
  };
}

// Component Props
interface SlideCreatorProps {
  topic: string;
  referenceDocuments?: string[];
  onPresentationGenerated: (presentation: Presentation) => void;
  onExport: () => Promise<Blob>;
}
```

### Backend API Endpoints

#### Authentication Module

```typescript
POST /api/auth/login
  Body: { email: string, password: string }
  Response: { accessToken: string, refreshToken: string, user: User }

POST /api/auth/register
  Body: { email: string, password: string, role: 'student' | 'faculty' }
  Response: { user: User }

POST /api/auth/refresh
  Body: { refreshToken: string }
  Response: { accessToken: string }

POST /api/auth/logout
  Headers: { Authorization: Bearer <token> }
  Response: { success: boolean }
```

#### AI Tutor Module

```typescript
POST /api/tutor/chat
  Headers: { Authorization: Bearer <token> }
  Body: { sessionId?: string, message: string }
  Response: { sessionId: string, response: ChatMessage, validated: boolean }

GET /api/tutor/sessions
  Headers: { Authorization: Bearer <token> }
  Response: { sessions: ChatSession[] }

GET /api/tutor/sessions/:id
  Headers: { Authorization: Bearer <token> }
  Response: { session: ChatSession }

DELETE /api/tutor/sessions/:id
  Headers: { Authorization: Bearer <token> }
  Response: { success: boolean }
```

#### Quiz Generator Module

```typescript
POST /api/quiz/generate
  Headers: { Authorization: Bearer <token> }
  Body: { topic?: string, documentId?: string, questionCount: number }
  Response: { session: QuizSession }

POST /api/quiz/answer
  Headers: { Authorization: Bearer <token> }
  Body: { sessionId: string, questionId: string, answer: number }
  Response: { correct: boolean, explanation: string, score: number }

GET /api/quiz/sessions/:id
  Headers: { Authorization: Bearer <token> }
  Response: { session: QuizSession }
```

#### Mind Map Module

```typescript
POST /api/mindmap/generate
  Headers: { Authorization: Bearer <token> }
  Body: { topic: string, depth?: number }
  Response: { mindMap: MindMap }

GET /api/mindmap/:id
  Headers: { Authorization: Bearer <token> }
  Response: { mindMap: MindMap }

POST /api/mindmap/:id/export
  Headers: { Authorization: Bearer <token> }
  Body: { format: 'png' | 'svg' }
  Response: Blob (image file)
```

#### Infographic Module

```typescript
POST /api/infographic/generate
  Headers: { Authorization: Bearer <token> }
  Body: { topic: string, type: InfographicType, data?: Record<string, any> }
  Response: { infographic: Infographic }

POST /api/infographic/:id/export
  Headers: { Authorization: Bearer <token> }
  Body: { format: 'png' | 'svg' }
  Response: Blob (image file)
```

#### Slide Creator Module

```typescript
POST /api/slides/generate
  Headers: { Authorization: Bearer <token> }
  Body: { topic: string, referenceDocuments?: string[], slideCount?: number }
  Response: { presentation: Presentation }

POST /api/slides/:id/export
  Headers: { Authorization: Bearer <token> }
  Response: Blob (PPTX file)
```

#### Document Processing Module

```typescript
POST /api/documents/upload
  Headers: { Authorization: Bearer <token> }
  Body: FormData { file: File }
  Response: { documentId: string, pageCount: number, status: 'processing' | 'ready' }

GET /api/documents/:id/status
  Headers: { Authorization: Bearer <token> }
  Response: { status: 'processing' | 'ready' | 'failed', progress: number }

POST /api/documents/:id/summarize
  Headers: { Authorization: Bearer <token> }
  Response: { summary: string, keyPoints: string[] }

POST /api/documents/:id/chat
  Headers: { Authorization: Bearer <token> }
  Body: { question: string }
  Response: { answer: string, citations: { page: number, text: string }[] }

DELETE /api/documents/:id
  Headers: { Authorization: Bearer <token> }
  Response: { success: boolean }
```

### Backend Services

#### Gemini Service

```typescript
interface GeminiService {
  // Generate chat response with context
  generateChatResponse(
    message: string,
    context: string[],
    systemPrompt: string
  ): Promise<string>;

  // Generate quiz questions
  generateQuizQuestions(
    topic: string,
    count: number,
    difficulty: string[]
  ): Promise<QuizQuestion[]>;

  // Generate quiz from document
  generateQuizFromDocument(
    documentText: string,
    count: number
  ): Promise<QuizQuestion[]>;

  // Generate mind map structure
  generateMindMap(
    topic: string,
    depth: number
  ): Promise<{ nodes: MindMapNode[], edges: MindMapEdge[] }>;

  // Generate infographic data
  generateInfographicData(
    topic: string,
    type: InfographicType
  ): Promise<InfographicData>;

  // Generate presentation slides
  generatePresentation(
    topic: string,
    referenceText?: string,
    slideCount?: number
  ): Promise<Slide[]>;

  // Answer question about document
  answerDocumentQuestion(
    question: string,
    documentChunks: string[]
  ): Promise<{ answer: string, relevantChunks: number[] }>;
}
```

#### Guard Agent Service

```typescript
interface ValidationResult {
  isValid: boolean;
  confidence: number;
  issues: string[];
  correctedContent?: string;
}

interface GuardAgentService {
  // Validate medical accuracy of AI response
  validateMedicalContent(
    content: string,
    context: string
  ): Promise<ValidationResult>;

  // Check for harmful content
  checkContentSafety(
    content: string
  ): Promise<{ isSafe: boolean, reasons: string[] }>;

  // Verify educational appropriateness
  verifyEducationalContent(
    content: string,
    userRole: 'student' | 'faculty'
  ): Promise<boolean>;

  // Log validation event
  logValidation(
    userId: string,
    content: string,
    result: ValidationResult
  ): Promise<void>;
}
```

#### Vector Store Service (Qdrant)

```typescript
interface DocumentChunk {
  id: string;
  documentId: string;
  text: string;
  pageNumber: number;
  embedding: number[];
  metadata: Record<string, any>;
}

interface VectorStoreService {
  // Store document embeddings
  storeDocumentEmbeddings(
    documentId: string,
    chunks: Omit<DocumentChunk, 'id' | 'embedding'>[]
  ): Promise<void>;

  // Search for relevant chunks
  searchSimilarChunks(
    query: string,
    documentId: string,
    limit: number
  ): Promise<DocumentChunk[]>;

  // Delete document embeddings
  deleteDocumentEmbeddings(
    documentId: string
  ): Promise<void>;
}
```

#### Document Processing Service

```typescript
interface DocumentProcessingService {
  // Extract text from PDF
  extractTextFromPDF(
    file: Buffer
  ): Promise<{ text: string, pageCount: number, pages: string[] }>;

  // Perform OCR on image
  performOCR(
    image: Buffer
  ): Promise<string>;

  // Chunk document for vector storage
  chunkDocument(
    text: string,
    chunkSize: number,
    overlap: number
  ): Promise<string[]>;

  // Generate document summary
  generateSummary(
    documentText: string
  ): Promise<{ summary: string, keyPoints: string[] }>;
}
```

## Data Models

### User Model

```typescript
interface User {
  id: string;
  email: string;
  passwordHash: string;
  role: 'student' | 'faculty';
  profile: {
    firstName: string;
    lastName: string;
    institution?: string;
    specialization?: string;
  };
  preferences: {
    focusMode: boolean;
    theme: 'light' | 'dark';
    notificationsEnabled: boolean;
  };
  createdAt: Date;
  updatedAt: Date;
  lastLoginAt: Date;
}
```

### Session Model

```typescript
interface Session {
  id: string;
  userId: string;
  accessToken: string;
  refreshToken: string;
  expiresAt: Date;
  createdAt: Date;
  lastActivityAt: Date;
  ipAddress: string;
  userAgent: string;
}
```

### Chat History Model

```typescript
interface ChatHistory {
  id: string;
  userId: string;
  sessionId: string;
  messages: ChatMessage[];
  context: string[];
  metadata: {
    totalMessages: number;
    averageResponseTime: number;
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### Document Model

```typescript
interface Document {
  id: string;
  userId: string;
  fileName: string;
  fileSize: number;
  mimeType: string;
  storageKey: string;
  status: 'uploading' | 'processing' | 'ready' | 'failed';
  processingProgress: number;
  metadata: {
    pageCount?: number;
    extractedText?: string;
    summary?: string;
    keyPoints?: string[];
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### Quiz History Model

```typescript
interface QuizHistory {
  id: string;
  userId: string;
  sessionId: string;
  topic: string;
  documentId?: string;
  questions: QuizQuestion[];
  answers: Map<string, number>;
  score: number;
  completedAt?: Date;
  createdAt: Date;
}
```

### Audit Log Model

```typescript
interface AuditLog {
  id: string;
  userId: string;
  action: string;
  resource: string;
  resourceId: string;
  details: Record<string, any>;
  ipAddress: string;
  userAgent: string;
  timestamp: Date;
}
```

### Guard Agent Log Model

```typescript
interface GuardAgentLog {
  id: string;
  userId: string;
  contentType: 'chat' | 'quiz' | 'document';
  originalContent: string;
  validationResult: ValidationResult;
  action: 'approved' | 'corrected' | 'blocked';
  timestamp: Date;
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

