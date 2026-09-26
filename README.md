# DevPilot

DevPilot is an AI-powered codebase assistant that lets you connect your GitHub repositories, index their source code, and ask questions about the codebase using retrieval-augmented generation (RAG). Answers are streamed in real time and include source-file citations to help developers understand and navigate their repositories.

## Features

- Sign in with GitHub OAuth
- Import public and private GitHub repositories
- Index repository files using code-aware chunking
- Store embeddings in PostgreSQL with `pgvector`
- Ask questions about an indexed repository
- Stream AI responses over Server-Sent Events
- Display source citations for generated answers
- Track repository indexing progress
- Create and manage repository-specific chat sessions
- Responsive dashboard and chat interface
- Light and dark theme support

## Architecture

DevPilot is composed of a Next.js frontend, a Spring Boot backend, and a PostgreSQL database with vector-search support.

```text
GitHub OAuth
     │
     ▼
Next.js client ───────► Spring Boot API
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
       GitHub API       PostgreSQL         OpenAI API
                              │
                              ▼
                         pgvector RAG
```

When a user selects a repository, the backend retrieves its files from GitHub, filters supported code files, splits them into chunks, generates embeddings, and stores them in PostgreSQL. During a chat session, relevant code chunks are retrieved and provided to the AI model as context. The response is then streamed back to the browser with citations.

## Tech Stack

### Frontend

- TypeScript
- Next.js 16 App Router
- React 19
- TanStack React Query
- Tailwind CSS
- shadcn/ui
- Lucide React
- Server-Sent Events for streamed chat responses

### Backend

- Java 21
- Spring Boot
- Spring MVC
- Spring Security
- GitHub OAuth 2.0
- Spring Data JPA
- Spring AI
- OpenAI integration
- PostgreSQL with pgvector
- Lombok

### Infrastructure

- Docker Compose
- PostgreSQL 16
- `pgvector`
- `hstore`
- `uuid-ossp`

## Repository Structure

```text
.
├── backend/
│   ├── pom.xml
│   └── src/
│       ├── main/java/devPilot/backend/
│       │   ├── config/          Application, CORS, security, and crypto configuration
│       │   ├── controllers/     Authentication, repository, and chat endpoints
│       │   ├── dto/             API request and response models
│       │   ├── entity/          JPA entities for users, repositories, and chats
│       │   ├── exceptions/      API exception types and error handling
│       │   ├── repository/      Spring Data repository interfaces
│       │   ├── security/        GitHub OAuth and authenticated-user handling
│       │   └── services/
│       │       ├── ai/           RAG retrieval, prompt construction, streaming, and citations
│       │       ├── github/       GitHub API integration and rate limiting
│       │       └── indexing/     File filtering, code chunking, and repository indexing
│       └── test/                Backend tests
├── client/
│   ├── app/
│   │   ├── auth/                Authentication callback routes
│   │   ├── chat/[repoId]/       Repository chat page
│   │   ├── dashboard/           Repository dashboard and settings pages
│   │   ├── login/               Login page
│   │   └── page.tsx             Landing page
│   ├── components/
│   │   ├── chat/                Chat view, messages, composer, and citations
│   │   ├── dashboard/           Repository cards, indexing state, and dashboards
│   │   ├── layout/              Application shell and navigation
│   │   ├── providers/           Authentication, theme, and query providers
│   │   └── ui/                  Shared interface components
│   ├── hooks/
│   │   ├── use-auth.ts          Authentication state
│   │   ├── use-chat.ts          Chat sessions and streamed messages
│   │   └── use-repos.ts         Repository loading and indexing state
│   ├── lib/
│   │   ├── api.ts               Typed backend API client
│   │   ├── stream-chat.ts       SSE chat response parser
│   │   └── query-keys.ts        React Query cache keys
│   └── package.json
├── docker/
│   └── postgres/
│       └── init-extensions.sql  PostgreSQL extensions required by DevPilot
└── docker-compose.yml            Local PostgreSQL service
```

## Prerequisites

Install the following before running DevPilot locally:

- Java 21
- Node.js and npm
- Docker and Docker Compose
- A GitHub OAuth application
- An OpenAI API key

## Local Development

### 1. Clone the repository

```bash
git clone https://github.com/Aestheticsuraj234/devPilot.git
cd devPilot
```

### 2. Start PostgreSQL

The Docker Compose configuration starts PostgreSQL with the required vector extensions.

```bash
docker compose up -d postgres
```

The database is exposed on port `5433` on the host and uses the following local development defaults:

```text
Database: devpilot
Username: postgres
Password: postgres
Host: localhost
Port: 5433
```

The initialization script enables:

- `vector`
- `hstore`
- `uuid-ossp`

### 3. Configure the backend

Configure the Spring Boot application with values for:

- PostgreSQL connection
- GitHub OAuth client credentials
- OpenAI API access
- Frontend URL
- Session and application security settings

The backend runs on port `8080` by default, which matches the frontend API client fallback.

GitHub OAuth should redirect users back to the frontend authentication callback after successful login. The backend protects `/api/**` endpoints and permits OAuth handshake routes.

### 4. Run the backend

```bash
cd backend
./mvnw spring-boot:run
```

On Windows:

```powershell
cd backend
mvnw.cmd spring-boot:run
```

The API should be available at:

```text
http://localhost:8080
```

### 5. Run the frontend

In another terminal:

```bash
cd client
npm install
npm run dev
```

The frontend should be available at:

```text
http://localhost:3000
```

The frontend uses the following API base URL by default:

```text
http://localhost:8080
```

To use another backend URL, set:

```text
NEXT_PUBLIC_API_BASE_URL=http://your-backend-host:port
```

## Available Commands

### Frontend

Run commands from the `client` directory:

```bash
npm run dev       # Start the development server
npm run build     # Create a production build
npm run start     # Start the production server
npm run lint      # Run ESLint
```

### Backend

Run commands from the `backend` directory:

```bash
./mvnw spring-boot:run  # Start the Spring Boot application
./mvnw test             # Run backend tests
./mvnw package          # Build the backend
```

## Application Flow

### GitHub authentication

1. The user selects **Continue with GitHub**.
2. The frontend redirects to the backend OAuth endpoint.
3. Spring Security authenticates the user with GitHub.
4. The backend creates or updates the local user record.
5. The user is redirected to the frontend authentication callback.
6. Authenticated API requests use the application session.

### Repository indexing

1. The frontend loads the user's GitHub repositories.
2. The backend synchronizes repository metadata through the GitHub API.
3. The user starts indexing for a selected repository.
4. The backend downloads eligible source files.
5. `CodeFileFilter` removes unsupported files.
6. `CodeChunker` splits source files into searchable chunks.
7. Spring AI generates embeddings.
8. Embeddings and metadata are stored in PostgreSQL using pgvector.
9. The frontend polls the repository status until indexing is complete.

### RAG chat

1. A chat session is created for an indexed repository.
2. The user submits a question.
3. The backend retrieves relevant code chunks using `CodeContextRetriever`.
4. `ChatPromptBuilder` builds a prompt containing the repository context.
5. The AI response is streamed using `ChatStreamHandler`.
6. The frontend parses SSE events in `stream-chat.ts`.
7. The response and citations are rendered by the chat components.

The main backend chat flow is coordinated by `ChatService`:

```text
Validate session and repository
        │
        ▼
Persist user message
        │
        ▼
Retrieve relevant code context
        │
        ▼
Build system and user prompts
        │
        ▼
Stream AI response with citations
```

## API Areas

The backend exposes API endpoints for:

- Authentication and current-user information
- GitHub repository synchronization
- Repository indexing
- Indexing status
- Chat session creation and retrieval
- Chat message history
- Streaming assistant responses

The frontend communicates with the API through the typed client in:

```text
client/lib/api.ts
```

Streaming chat responses are handled in:

```text
client/lib/stream-chat.ts
```

## Database

DevPilot uses PostgreSQL with pgvector for storing repository data and vector embeddings.

The local Docker database is configured in `docker-compose.yml` and is initialized by:

```text
docker/postgres/init-extensions.sql
```

To stop the database:

```bash
docker compose down
```

To remove the local database volume as well:

```bash
docker compose down -v
```

## Security Notes

- Do not commit GitHub OAuth secrets or OpenAI API keys.
- Use environment-specific configuration for local and production deployments.
- The backend requires authentication for `/api/**` endpoints.
- Repository ownership is checked before accessing repository data or chat sessions.
- GitHub access tokens are handled by the backend rather than exposed to the frontend.

## Troubleshooting

### PostgreSQL is unavailable

Check the container status:

```bash
docker compose ps
```

View database logs:

```bash
docker compose logs postgres
```

### The frontend cannot connect to the backend

Confirm that the backend is running on port `8080`, or set:

```text
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080
```

### Chat cannot be started

A repository must finish indexing before a chat session can be created. Check the repository status in the dashboard and wait until it reaches `READY`.

### GitHub login fails

Verify that:

- The GitHub OAuth client ID and secret are configured.
- The callback URL matches the OAuth application configuration.
- The backend frontend URL points to the correct frontend origin.
- The frontend and backend are running on the expected ports.

## Project Status

DevPilot is an active development project. Configuration, API contracts, and deployment requirements may evolve as additional repository indexing and AI features are added.
