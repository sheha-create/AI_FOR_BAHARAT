# Design Document: CodeMentor AI

## Overview

CodeMentor AI is a cloud-based intelligent learning and developer productivity platform that combines adaptive learning, context-aware code assistance, and collaborative features. The system uses large language models (LLMs), static code analysis, and machine learning to provide personalized developer education and productivity tools.

### Design Goals

1. **Personalization**: Adapt to individual developer skill levels and learning styles
2. **Context Awareness**: Understand entire codebases, not just isolated code snippets
3. **Real-time Performance**: Respond to queries within 2 seconds for 95% of requests
4. **Scalability**: Support 100K+ concurrent users and codebases up to 1M lines
5. **Privacy**: Ensure user code is never used for training without consent
6. **Extensibility**: Support multiple languages, IDEs, and integration points

### High-Level Architecture

The system follows a microservices architecture with the following major components:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Web App  │  │IDE Plugin│  │  Mobile  │  │   API    │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│         (Authentication, Rate Limiting, Routing)                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Service Layer                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Learning   │  │     Code     │  │     User     │         │
│  │    Engine    │  │  Assistant   │  │   Service    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Knowledge   │  │  Analytics   │  │   Billing    │         │
│  │     Base     │  │   Service    │  │   Service    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Data Layer                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  PostgreSQL  │  │    Redis     │  │  Vector DB   │         │
│  │  (Metadata)  │  │   (Cache)    │  │ (Embeddings) │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │   S3/Blob    │  │  Elasticsearch│                            │
│  │ (Code Store) │  │   (Search)   │                            │
│  └──────────────┘  └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AI/ML Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  LLM Service │  │ Code Analysis│  │  Skill Model │         │
│  │  (GPT-4/etc) │  │    Engine    │  │   (Custom)   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

## Architecture

### System Components

#### 1. API Gateway

**Purpose**: Single entry point for all client requests, handling authentication, rate limiting, and routing.

**Technology**: Kong or AWS API Gateway

**Responsibilities**:
- JWT token validation and session management
- Rate limiting per user/tier (Free: 100 req/hour, Pro: 1000 req/hour, Team: 5000 req/hour)
- Request routing to appropriate microservices
- TLS termination and security headers
- Request/response logging and monitoring
- CORS handling for web clients

**Key Design Decisions**:
- Use JWT tokens with 1-hour expiration and refresh token mechanism
- Implement distributed rate limiting using Redis to handle multi-instance deployments
- Route based on URL path patterns and HTTP methods


#### 2. User Service

**Purpose**: Manages user accounts, authentication, profiles, and subscription status.

**Technology**: Node.js/TypeScript with Express

**Responsibilities**:
- User registration and authentication (email/password, OAuth, SSO)
- Password hashing using bcrypt with salt rounds = 12
- Profile management and preferences
- Subscription tier management and feature flags
- Account deletion and GDPR compliance
- Audit log generation for security events

**Database Schema**:
```typescript
interface User {
  id: string;              // UUID
  email: string;           // Unique, indexed
  passwordHash: string;    // bcrypt hash
  name: string;
  createdAt: Date;
  lastLoginAt: Date;
  subscriptionTier: 'free' | 'pro' | 'team' | 'enterprise';
  subscriptionExpiresAt: Date | null;
  preferences: UserPreferences;
  isActive: boolean;
  failedLoginAttempts: number;
  lockedUntil: Date | null;
}

interface UserPreferences {
  language: string;              // UI language
  theme: 'light' | 'dark' | 'auto';
  defaultExplanationDepth: 'eli5' | 'intermediate' | 'expert';
  enableSocraticMode: boolean;
  enableOfflineMode: boolean;
}
```


#### 3. Learning Engine Service

**Purpose**: Manages adaptive learning paths, skill assessment, and personalized content delivery.

**Technology**: Python with FastAPI (for ML integration)

**Responsibilities**:
- Skill profile management and updates
- Learning path generation and adaptation
- Confusion signal detection from user interactions
- Learning debt tracking
- Practice problem generation
- Progress tracking and analytics

**Key Algorithms**:

1. **Skill Assessment Algorithm**:
   - Bayesian Knowledge Tracing (BKT) to estimate skill mastery probability
   - Update skill scores based on: problem completion time, correctness, hint usage, confusion signals
   - Maintain skill graph with dependencies (e.g., "async/await" depends on "promises")

2. **Confusion Signal Detection**:
   - Time spent on explanation (>2x expected = confusion)
   - Re-reading same content multiple times
   - Requesting simpler explanation depth
   - Failed practice problems after explanation
   - Explicit "I don't understand" feedback

3. **Learning Path Adaptation**:
   - Use A* search to find optimal path from current skills to goal
   - Dynamically adjust based on performance (skip mastered content, add remedial content)
   - Balance breadth vs depth based on user goals


**Database Schema**:
```typescript
interface SkillProfile {
  userId: string;
  skills: Map<string, SkillScore>;  // skill_id -> score
  learningDebt: LearningDebtItem[];
  lastUpdated: Date;
}

interface SkillScore {
  skillId: string;           // e.g., "javascript.promises"
  masteryLevel: number;      // 0.0 to 1.0
  confidence: number;        // 0.0 to 1.0 (certainty of estimate)
  lastPracticed: Date;
  practiceCount: number;
  successRate: number;
}

interface LearningDebtItem {
  conceptId: string;
  usageCount: number;        // Times used without understanding
  severity: 'low' | 'medium' | 'high';
  detectedAt: Date;
}

interface LearningPath {
  id: string;
  userId: string;
  goal: string;              // e.g., "Learn React Hooks"
  currentStep: number;
  steps: LearningStep[];
  status: 'active' | 'completed' | 'abandoned';
  createdAt: Date;
  completedAt: Date | null;
}

interface LearningStep {
  stepNumber: number;
  conceptId: string;
  contentType: 'explanation' | 'practice' | 'project';
  estimatedDuration: number; // minutes
  completed: boolean;
  performance: number | null; // 0.0 to 1.0
}
```


#### 4. Code Assistant Service

**Purpose**: Provides context-aware code generation, analysis, refactoring, and documentation.

**Technology**: Python with FastAPI

**Responsibilities**:
- Codebase indexing and context building
- Code generation with style matching
- Multi-file refactoring analysis
- Documentation generation
- Dependency analysis
- Integration with static analysis tools

**Codebase Analysis Pipeline**:

1. **Initial Indexing**:
   - Parse all source files using language-specific parsers (tree-sitter)
   - Extract AST, symbols, imports, and dependencies
   - Detect coding style (indentation, naming conventions, patterns)
   - Build dependency graph
   - Generate embeddings for semantic search
   - Store in vector database for similarity search

2. **Incremental Updates**:
   - Watch for file changes via IDE integration or Git hooks
   - Re-parse only changed files and their dependents
   - Update indexes and embeddings incrementally
   - Invalidate affected cache entries

3. **Context Retrieval**:
   - For code generation: retrieve similar code patterns, relevant imports, style rules
   - For explanations: retrieve related functions, type definitions, documentation
   - Use hybrid search (keyword + semantic) for best results


**Database Schema**:
```typescript
interface CodebaseContext {
  id: string;
  userId: string;
  projectName: string;
  repositoryUrl: string | null;
  primaryLanguage: string;
  languages: string[];       // All detected languages
  framework: string[];       // Detected frameworks
  dependencies: Dependency[];
  styleGuide: StyleGuide;
  indexedAt: Date;
  lastUpdatedAt: Date;
  lineCount: number;
  fileCount: number;
}

interface Dependency {
  name: string;
  version: string;
  type: 'production' | 'development';
  packageManager: string;    // npm, pip, maven, etc.
}

interface StyleGuide {
  indentation: 'spaces' | 'tabs';
  indentSize: number;
  namingConventions: {
    functions: 'camelCase' | 'snake_case' | 'PascalCase';
    classes: 'PascalCase' | 'snake_case';
    constants: 'UPPER_SNAKE_CASE' | 'camelCase';
  };
  maxLineLength: number;
  quoteStyle: 'single' | 'double';
  semicolons: boolean;       // For JavaScript/TypeScript
}

interface CodeEmbedding {
  id: string;
  codebaseId: string;
  filePath: string;
  functionName: string | null;
  codeSnippet: string;
  embedding: number[];       // 1536-dim vector for OpenAI embeddings
  language: string;
  createdAt: Date;
}
```


#### 5. Knowledge Base Service

**Purpose**: Manages team knowledge bases, documentation, and shared learning resources.

**Technology**: Python with FastAPI

**Responsibilities**:
- Team knowledge base creation and management
- Knowledge entry curation and search
- Team-specific pattern extraction
- Expertise mapping (who knows what)
- Onboarding content generation

**Database Schema**:
```typescript
interface TeamKnowledgeBase {
  id: string;
  teamId: string;
  entries: KnowledgeEntry[];
  patterns: TeamPattern[];
  expertiseMap: Map<string, string[]>; // userId -> skill areas
  createdAt: Date;
}

interface KnowledgeEntry {
  id: string;
  title: string;
  content: string;
  category: string;
  tags: string[];
  authorId: string;
  createdAt: Date;
  updatedAt: Date;
  upvotes: number;
  relatedCodePaths: string[];
}

interface TeamPattern {
  id: string;
  name: string;
  description: string;
  codeExample: string;
  usageCount: number;
  category: 'architecture' | 'style' | 'convention' | 'antipattern';
}
```


#### 6. Analytics Service

**Purpose**: Tracks user interactions, learning progress, and system metrics.

**Technology**: Python with FastAPI, Apache Kafka for event streaming

**Responsibilities**:
- Event ingestion and processing
- Progress tracking and visualization
- A/B testing framework
- System performance monitoring
- Usage analytics for billing

**Event Types**:
- User interactions (code generation, explanations, debugging sessions)
- Learning events (practice problems, skill assessments, path completions)
- System events (API calls, errors, latency metrics)
- Business events (subscriptions, upgrades, feature usage)

#### 7. Billing Service

**Purpose**: Manages subscriptions, payments, and usage tracking.

**Technology**: Node.js/TypeScript with Stripe integration

**Responsibilities**:
- Subscription lifecycle management
- Payment processing via Stripe
- Invoice generation
- Usage-based billing calculations
- Quota enforcement


#### 8. LLM Service

**Purpose**: Interfaces with large language models for code generation, explanations, and natural language understanding.

**Technology**: Python with LangChain/LlamaIndex

**Responsibilities**:
- Prompt engineering and management
- LLM API calls with retry logic and fallbacks
- Response parsing and validation
- Context window management (truncation, summarization)
- Multi-model support (GPT-4, Claude, Llama, etc.)
- Cost optimization and caching

**Prompt Templates**:

1. **Code Generation Prompt**:
```
You are a code generation assistant. Generate code that matches the following context:

Language: {language}
Framework: {framework}
Style Guide:
- Indentation: {indentation}
- Naming: {naming_conventions}
- Max line length: {max_line_length}

Existing code context:
{relevant_code_snippets}

User request: {user_request}

Generate code that:
1. Follows the existing style exactly
2. Uses only dependencies already in the project: {dependencies}
3. Integrates seamlessly with the provided context
4. Includes brief inline comments for complex logic

Code:
```

2. **Explanation Prompt**:
```
You are a programming tutor. Explain the following code to a developer with {skill_level} expertise in {language}.

Code:
{code_snippet}

Codebase context:
{relevant_context}

Provide an explanation at {explanation_depth} level:
- eli5: Use simple analogies, avoid jargon
- intermediate: Assume basic programming knowledge
- expert: Use technical terminology, focus on nuances

Explanation:
```


3. **Socratic Debugging Prompt**:
```
You are a debugging coach using the Socratic method. A developer is stuck on the following problem:

Error: {error_message}
Code: {code_snippet}
Context: {codebase_context}

Instead of providing direct solutions, ask clarifying questions that guide the developer to discover the solution themselves. Questions should:
1. Help them understand what the code is actually doing
2. Identify assumptions that might be wrong
3. Suggest systematic debugging approaches
4. Build problem-solving skills

Current conversation:
{conversation_history}

Your next question:
```

**LLM Response Caching Strategy**:
- Cache responses for identical prompts for 24 hours
- Use semantic similarity for near-duplicate prompts (cosine similarity > 0.95)
- Invalidate cache when codebase context changes
- Store cache in Redis with LRU eviction


#### 9. Code Analysis Engine

**Purpose**: Performs static analysis, linting, and security scanning independent of LLMs.

**Technology**: Python with language-specific analyzers

**Responsibilities**:
- AST parsing using tree-sitter (supports 40+ languages)
- Static analysis for code quality issues
- Security vulnerability detection (OWASP Top 10)
- Complexity metrics (cyclomatic complexity, cognitive complexity)
- Code smell detection
- Dependency vulnerability scanning

**Supported Analyzers**:
- JavaScript/TypeScript: ESLint, TypeScript compiler
- Python: Pylint, Bandit (security), mypy (types)
- Java: SpotBugs, PMD
- Go: golangci-lint
- Rust: Clippy
- Generic: SonarQube rules

**Analysis Output**:
```typescript
interface AnalysisResult {
  filePath: string;
  language: string;
  issues: Issue[];
  metrics: CodeMetrics;
  securityFindings: SecurityFinding[];
}

interface Issue {
  line: number;
  column: number;
  severity: 'error' | 'warning' | 'info';
  rule: string;
  message: string;
  suggestion: string | null;
}

interface CodeMetrics {
  linesOfCode: number;
  cyclomaticComplexity: number;
  cognitiveComplexity: number;
  maintainabilityIndex: number;
  testCoverage: number | null;
}

interface SecurityFinding {
  type: string;              // e.g., "SQL Injection", "XSS"
  severity: 'critical' | 'high' | 'medium' | 'low';
  line: number;
  description: string;
  remediation: string;
  cweId: string | null;
}
```


## Components and Interfaces

### API Specifications

#### REST API Endpoints

**Authentication Endpoints**:
```
POST /api/v1/auth/register
Request: { email, password, name }
Response: { userId, token, refreshToken }

POST /api/v1/auth/login
Request: { email, password }
Response: { userId, token, refreshToken, subscriptionTier }

POST /api/v1/auth/refresh
Request: { refreshToken }
Response: { token, refreshToken }

POST /api/v1/auth/logout
Request: { token }
Response: { success: true }
```

**Code Assistant Endpoints**:
```
POST /api/v1/code/analyze
Request: { codebaseId?, files: [{ path, content }], language }
Response: { analysisResults: AnalysisResult[] }

POST /api/v1/code/generate
Request: { 
  codebaseId, 
  prompt, 
  language, 
  context: { filePath, cursorPosition, surroundingCode }
}
Response: { 
  generatedCode, 
  explanation, 
  confidence: number 
}

POST /api/v1/code/explain
Request: { 
  code, 
  language, 
  explanationDepth: 'eli5' | 'intermediate' | 'expert',
  codebaseId?
}
Response: { 
  explanation, 
  keyPoints: string[],
  relatedConcepts: string[],
  visualizations?: Diagram[]
}

POST /api/v1/code/refactor
Request: { 
  codebaseId, 
  filePaths: string[], 
  refactoringType: string,
  targetPattern?: string
}
Response: { 
  changes: FileChange[],
  affectedFiles: string[],
  breakingChanges: BreakingChange[],
  testSuggestions: string[]
}
```


**Learning Engine Endpoints**:
```
GET /api/v1/learning/profile
Response: { skillProfile: SkillProfile }

POST /api/v1/learning/assess
Request: { 
  skillArea: string,
  responses: AssessmentResponse[]
}
Response: { 
  updatedSkills: SkillScore[],
  recommendations: string[]
}

POST /api/v1/learning/path/create
Request: { 
  goal: string,
  timeCommitment: number,  // hours per week
  currentSkills?: string[]
}
Response: { 
  learningPath: LearningPath,
  estimatedDuration: number
}

GET /api/v1/learning/path/{pathId}
Response: { learningPath: LearningPath }

POST /api/v1/learning/path/{pathId}/progress
Request: { 
  stepNumber: number,
  completed: boolean,
  performance?: number,
  timeSpent: number
}
Response: { 
  updatedPath: LearningPath,
  nextStep: LearningStep,
  skillUpdates: SkillScore[]
}

POST /api/v1/learning/practice/generate
Request: { 
  skillArea: string,
  difficulty: number,
  type: 'coding' | 'multiple-choice' | 'debugging'
}
Response: { 
  problem: PracticeProblem,
  testCases?: TestCase[]
}

POST /api/v1/learning/practice/submit
Request: { 
  problemId: string,
  solution: string,
  language: string
}
Response: { 
  correct: boolean,
  feedback: string,
  testResults: TestResult[],
  alternativeApproaches?: string[]
}
```


**Debugging Endpoints**:
```
POST /api/v1/debug/translate-error
Request: { 
  errorMessage: string,
  stackTrace?: string,
  code?: string,
  language: string,
  codebaseId?
}
Response: { 
  plainEnglish: string,
  possibleCauses: Cause[],
  solutions: Solution[],
  diagnosticSteps: string[]
}

POST /api/v1/debug/socratic
Request: { 
  conversationId?: string,
  userMessage: string,
  code?: string,
  error?: string
}
Response: { 
  conversationId: string,
  question: string,
  hints?: string[],
  progressAssessment: number  // 0-1, how close to solution
}
```

**Codebase Management Endpoints**:
```
POST /api/v1/codebase/create
Request: { 
  name: string,
  repositoryUrl?: string,
  primaryLanguage: string
}
Response: { 
  codebaseId: string,
  status: 'created'
}

POST /api/v1/codebase/{id}/upload
Request: FormData with files
Response: { 
  uploadedFiles: number,
  status: 'indexing'
}

GET /api/v1/codebase/{id}/status
Response: { 
  status: 'indexing' | 'ready' | 'error',
  progress: number,
  lineCount: number,
  fileCount: number,
  indexedAt?: Date
}

POST /api/v1/codebase/{id}/sync
Request: { 
  changes: FileChange[]
}
Response: { 
  status: 'synced',
  updatedFiles: number
}

DELETE /api/v1/codebase/{id}
Response: { success: true }
```


**Team Endpoints**:
```
POST /api/v1/team/create
Request: { name: string, memberEmails: string[] }
Response: { teamId: string, invitesSent: number }

POST /api/v1/team/{id}/knowledge/add
Request: { 
  title: string,
  content: string,
  category: string,
  tags: string[]
}
Response: { entryId: string }

GET /api/v1/team/{id}/knowledge/search
Query: { q: string, category?: string, tags?: string[] }
Response: { 
  entries: KnowledgeEntry[],
  totalResults: number
}

GET /api/v1/team/{id}/expertise
Response: { 
  expertiseMap: Map<string, string[]>,
  skillGaps: string[]
}
```

### WebSocket API

For real-time features (live debugging sessions, collaborative learning):

```
WS /api/v1/ws/debug/{sessionId}
Messages:
  - user_message: { type: 'message', content: string }
  - assistant_response: { type: 'response', content: string, hints?: string[] }
  - code_update: { type: 'code', content: string }
  - session_end: { type: 'end', summary: string }

WS /api/v1/ws/collab/{roomId}
Messages:
  - join: { type: 'join', userId: string }
  - code_share: { type: 'code', content: string, language: string }
  - cursor_position: { type: 'cursor', line: number, column: number }
  - leave: { type: 'leave', userId: string }
```


### IDE Plugin Architecture

**Supported IDEs**:
- Visual Studio Code (TypeScript extension)
- IntelliJ IDEA / PyCharm (Kotlin/Java plugin)
- Visual Studio (C# extension)

**Plugin Components**:

1. **Language Server Protocol (LSP) Integration**:
   - Implement custom LSP server for CodeMentor features
   - Provide code completions, hover information, diagnostics
   - Integrate with existing IDE features seamlessly

2. **UI Components**:
   - Sidebar panel for explanations and learning content
   - Inline code actions for generation and refactoring
   - Status bar indicators for analysis progress
   - Notification system for learning recommendations

3. **Local Cache**:
   - Cache recent explanations and code snippets
   - Store user preferences and session state
   - Enable offline mode with cached content

4. **Communication**:
   - REST API calls for most operations
   - WebSocket for real-time debugging sessions
   - Batch requests to minimize network overhead

**VS Code Extension Example**:
```typescript
// Extension activation
export function activate(context: vscode.ExtensionContext) {
  // Register commands
  context.subscriptions.push(
    vscode.commands.registerCommand('codementor.explain', explainCode),
    vscode.commands.registerCommand('codementor.generate', generateCode),
    vscode.commands.registerCommand('codementor.debug', startDebugSession)
  );
  
  // Register code action provider
  context.subscriptions.push(
    vscode.languages.registerCodeActionsProvider(
      { scheme: 'file' },
      new CodeMentorActionProvider()
    )
  );
  
  // Initialize API client
  const apiClient = new CodeMentorAPI(getApiKey());
  
  // Start file watcher for codebase sync
  const watcher = vscode.workspace.createFileSystemWatcher('**/*');
  watcher.onDidChange(uri => syncFileChange(uri, apiClient));
}
```


## Data Models

### Core Data Structures

**User and Authentication**:
```typescript
interface User {
  id: string;
  email: string;
  passwordHash: string;
  name: string;
  createdAt: Date;
  lastLoginAt: Date;
  subscriptionTier: SubscriptionTier;
  subscriptionExpiresAt: Date | null;
  preferences: UserPreferences;
  isActive: boolean;
  failedLoginAttempts: number;
  lockedUntil: Date | null;
  teamId: string | null;
}

type SubscriptionTier = 'free' | 'pro' | 'team' | 'enterprise';

interface Session {
  id: string;
  userId: string;
  token: string;
  refreshToken: string;
  expiresAt: Date;
  createdAt: Date;
  ipAddress: string;
  userAgent: string;
}
```

**Learning Models**:
```typescript
interface SkillProfile {
  userId: string;
  skills: Map<string, SkillScore>;
  learningDebt: LearningDebtItem[];
  lastUpdated: Date;
}

interface SkillScore {
  skillId: string;
  masteryLevel: number;      // 0.0 to 1.0
  confidence: number;        // 0.0 to 1.0
  lastPracticed: Date;
  practiceCount: number;
  successRate: number;
  relatedSkills: string[];   // Dependencies
}

interface PracticeProblem {
  id: string;
  skillArea: string;
  difficulty: number;        // 1-10
  type: 'coding' | 'multiple-choice' | 'debugging';
  title: string;
  description: string;
  starterCode?: string;
  testCases: TestCase[];
  hints: string[];
  solution: string;
  explanation: string;
}

interface TestCase {
  input: any;
  expectedOutput: any;
  isHidden: boolean;        // Hidden test cases for validation
}
```


**Code Models**:
```typescript
interface CodebaseContext {
  id: string;
  userId: string;
  projectName: string;
  repositoryUrl: string | null;
  primaryLanguage: string;
  languages: string[];
  frameworks: string[];
  dependencies: Dependency[];
  styleGuide: StyleGuide;
  indexedAt: Date;
  lastUpdatedAt: Date;
  lineCount: number;
  fileCount: number;
  status: 'indexing' | 'ready' | 'error';
}

interface FileChange {
  path: string;
  type: 'added' | 'modified' | 'deleted';
  content?: string;
  oldContent?: string;
}

interface RefactoringResult {
  changes: FileChange[];
  affectedFiles: string[];
  breakingChanges: BreakingChange[];
  testSuggestions: string[];
  estimatedRisk: 'low' | 'medium' | 'high';
}

interface BreakingChange {
  type: 'api' | 'signature' | 'behavior';
  location: string;
  description: string;
  affectedCallers: string[];
}
```

**Debugging Models**:
```typescript
interface DebugSession {
  id: string;
  userId: string;
  mode: 'socratic' | 'direct';
  startedAt: Date;
  endedAt: Date | null;
  messages: DebugMessage[];
  problem: string;
  solution: string | null;
  learningPoints: string[];
}

interface DebugMessage {
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  codeSnapshot?: string;
}

interface ErrorTranslation {
  originalError: string;
  plainEnglish: string;
  possibleCauses: Cause[];
  solutions: Solution[];
  diagnosticSteps: string[];
}

interface Cause {
  description: string;
  likelihood: number;       // 0.0 to 1.0
  indicators: string[];
}

interface Solution {
  description: string;
  codeExample?: string;
  rank: number;             // 1 = most likely to work
  difficulty: 'easy' | 'medium' | 'hard';
  estimatedTime: string;
}
```


**Team Models**:
```typescript
interface Team {
  id: string;
  name: string;
  createdAt: Date;
  subscriptionTier: SubscriptionTier;
  memberIds: string[];
  knowledgeBaseId: string;
  settings: TeamSettings;
}

interface TeamSettings {
  allowExternalSharing: boolean;
  requireCodeReview: boolean;
  defaultPrivacy: 'private' | 'team' | 'public';
  ssoProvider?: string;
}

interface TeamKnowledgeBase {
  id: string;
  teamId: string;
  entries: KnowledgeEntry[];
  patterns: TeamPattern[];
  expertiseMap: Map<string, string[]>;
  createdAt: Date;
}
```

### Database Schema Design

**PostgreSQL Tables**:

```sql
-- Users and Authentication
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  last_login_at TIMESTAMP,
  subscription_tier VARCHAR(50) DEFAULT 'free',
  subscription_expires_at TIMESTAMP,
  is_active BOOLEAN DEFAULT true,
  failed_login_attempts INTEGER DEFAULT 0,
  locked_until TIMESTAMP,
  team_id UUID REFERENCES teams(id),
  preferences JSONB DEFAULT '{}'
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_team ON users(team_id);

CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  token VARCHAR(512) UNIQUE NOT NULL,
  refresh_token VARCHAR(512) UNIQUE NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  ip_address INET,
  user_agent TEXT
);

CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_token ON sessions(token);
```


```sql
-- Learning Data
CREATE TABLE skill_profiles (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  skills JSONB NOT NULL DEFAULT '{}',
  learning_debt JSONB DEFAULT '[]',
  last_updated TIMESTAMP DEFAULT NOW()
);

CREATE TABLE learning_paths (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  goal TEXT NOT NULL,
  current_step INTEGER DEFAULT 0,
  steps JSONB NOT NULL,
  status VARCHAR(50) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);

CREATE INDEX idx_learning_paths_user ON learning_paths(user_id);
CREATE INDEX idx_learning_paths_status ON learning_paths(status);

CREATE TABLE practice_problems (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  skill_area VARCHAR(255) NOT NULL,
  difficulty INTEGER CHECK (difficulty BETWEEN 1 AND 10),
  type VARCHAR(50) NOT NULL,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  starter_code TEXT,
  test_cases JSONB NOT NULL,
  hints JSONB DEFAULT '[]',
  solution TEXT NOT NULL,
  explanation TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_practice_problems_skill ON practice_problems(skill_area);
CREATE INDEX idx_practice_problems_difficulty ON practice_problems(difficulty);

CREATE TABLE practice_submissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  problem_id UUID REFERENCES practice_problems(id),
  solution TEXT NOT NULL,
  language VARCHAR(50) NOT NULL,
  correct BOOLEAN NOT NULL,
  test_results JSONB,
  submitted_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_practice_submissions_user ON practice_submissions(user_id);
CREATE INDEX idx_practice_submissions_problem ON practice_submissions(problem_id);
```


```sql
-- Codebase Data
CREATE TABLE codebases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  project_name VARCHAR(255) NOT NULL,
  repository_url TEXT,
  primary_language VARCHAR(50) NOT NULL,
  languages TEXT[] DEFAULT '{}',
  frameworks TEXT[] DEFAULT '{}',
  dependencies JSONB DEFAULT '[]',
  style_guide JSONB,
  indexed_at TIMESTAMP,
  last_updated_at TIMESTAMP DEFAULT NOW(),
  line_count INTEGER DEFAULT 0,
  file_count INTEGER DEFAULT 0,
  status VARCHAR(50) DEFAULT 'created'
);

CREATE INDEX idx_codebases_user ON codebases(user_id);
CREATE INDEX idx_codebases_status ON codebases(status);

-- Code embeddings stored in vector database (Pinecone/Weaviate)
-- Not in PostgreSQL due to vector search requirements

-- Debug Sessions
CREATE TABLE debug_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  mode VARCHAR(50) NOT NULL,
  started_at TIMESTAMP DEFAULT NOW(),
  ended_at TIMESTAMP,
  messages JSONB NOT NULL DEFAULT '[]',
  problem TEXT NOT NULL,
  solution TEXT,
  learning_points JSONB DEFAULT '[]'
);

CREATE INDEX idx_debug_sessions_user ON debug_sessions(user_id);
CREATE INDEX idx_debug_sessions_started ON debug_sessions(started_at);
```


```sql
-- Team Data
CREATE TABLE teams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  subscription_tier VARCHAR(50) DEFAULT 'team',
  settings JSONB DEFAULT '{}'
);

CREATE TABLE team_knowledge_bases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_id UUID UNIQUE REFERENCES teams(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE knowledge_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  knowledge_base_id UUID REFERENCES team_knowledge_bases(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  category VARCHAR(100),
  tags TEXT[] DEFAULT '{}',
  author_id UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  upvotes INTEGER DEFAULT 0,
  related_code_paths TEXT[] DEFAULT '{}'
);

CREATE INDEX idx_knowledge_entries_kb ON knowledge_entries(knowledge_base_id);
CREATE INDEX idx_knowledge_entries_category ON knowledge_entries(category);
CREATE INDEX idx_knowledge_entries_tags ON knowledge_entries USING GIN(tags);

-- Analytics Events
CREATE TABLE analytics_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  event_type VARCHAR(100) NOT NULL,
  event_data JSONB NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_analytics_events_user ON analytics_events(user_id);
CREATE INDEX idx_analytics_events_type ON analytics_events(event_type);
CREATE INDEX idx_analytics_events_created ON analytics_events(created_at);
```


### Redis Cache Structure

```
# Session cache
session:{token} -> { userId, expiresAt, subscriptionTier }
TTL: 1 hour

# Rate limiting
ratelimit:{userId}:{endpoint} -> request_count
TTL: 1 hour

# LLM response cache
llm:response:{prompt_hash} -> { response, timestamp }
TTL: 24 hours

# Codebase context cache
codebase:{codebaseId}:context -> { styleGuide, dependencies, patterns }
TTL: 1 hour

# User preferences cache
user:{userId}:prefs -> UserPreferences
TTL: 1 hour

# Skill profile cache
user:{userId}:skills -> SkillProfile
TTL: 15 minutes
```

### Vector Database (Pinecone/Weaviate)

```typescript
// Code embedding schema
interface CodeEmbeddingVector {
  id: string;                    // Unique identifier
  vector: number[];              // 1536-dim embedding
  metadata: {
    codebaseId: string;
    userId: string;
    filePath: string;
    functionName: string | null;
    language: string;
    codeSnippet: string;         // Original code (max 2000 chars)
    lineStart: number;
    lineEnd: number;
    createdAt: string;
  };
}

// Search query
interface CodeSearchQuery {
  vector: number[];              // Query embedding
  topK: number;                  // Number of results
  filter: {
    codebaseId?: string;
    language?: string;
    userId?: string;
  };
}
```


### Elasticsearch Schema

```json
{
  "mappings": {
    "properties": {
      "codebaseId": { "type": "keyword" },
      "userId": { "type": "keyword" },
      "filePath": { "type": "keyword" },
      "language": { "type": "keyword" },
      "content": { 
        "type": "text",
        "analyzer": "code_analyzer"
      },
      "functionName": { "type": "text" },
      "className": { "type": "text" },
      "imports": { "type": "keyword" },
      "dependencies": { "type": "keyword" },
      "lastModified": { "type": "date" }
    }
  },
  "settings": {
    "analysis": {
      "analyzer": {
        "code_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "code_stop"]
        }
      },
      "filter": {
        "code_stop": {
          "type": "stop",
          "stopwords": ["the", "a", "an"]
        }
      }
    }
  }
}
```

