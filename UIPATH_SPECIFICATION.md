# UiPath Implementation Specification for Onyx (Danswer)

## Overview

Onyx (formerly Danswer) is an enterprise AI platform that provides intelligent search and conversational AI capabilities by connecting to company documents, applications, and knowledge bases. The system ingests data from 40+ connectors (Google Drive, Slack, Confluence, etc.), processes and indexes content using vector embeddings and keyword search, and provides a chat interface powered by configurable LLMs. 

It implements RAG (Retrieval Augmented Generation) to ground AI responses in verified company data, includes permission management to ensure users only access authorized content, and supports both single-tenant and multi-tenant deployments with scalability from laptop to enterprise cloud infrastructure.

---

## Technical Flow

The following outlines the complete step-by-step process flow of the Onyx system:

### 1. User Authentication & Authorization
Users authenticate via configurable methods (OAuth2, SAML, OIDC, or basic auth). The system validates credentials, creates/retrieves user sessions with JWT tokens, and maintains RBAC permissions throughout the session.

### 2. Connector Configuration & Credential Management
Administrators configure data source connectors via the web UI, specifying connector type, API credentials, and sync parameters. Credentials are encrypted and stored in PostgreSQL. Each connector-credential pair (CC Pair) is created with specific access permissions.

### 3. Data Ingestion - Initial Load
Background job scheduler (Celery) triggers connector load operations. Each connector implements LoadConnector interface to fetch complete datasets from external sources (e.g., all Confluence pages, Slack messages, Google Drive files). Connectors make authenticated API calls to external systems using stored credentials.

### 4. Data Ingestion - Incremental Polling
After initial load, PollConnector interface enables time-based incremental updates. System polls sources at configured intervals, fetching only documents modified since last sync using timestamp filters. This reduces API calls and processing overhead.

### 5. Document Processing & Chunking
Raw documents flow through processing pipeline. HTML is parsed to extract text and metadata. Documents are split into semantic chunks (default 512 tokens) using intelligent chunking that preserves context. Each chunk maintains references to source document, creation date, and access permissions.

### 6. Embedding Generation
Chunks are batched (default 32 documents) and sent to the Model Server. The server runs the configured embedding model (default: multilingual-e5 models) to generate dense vector representations. Embeddings capture semantic meaning and enable similarity search. Batching optimizes GPU/CPU utilization.

### 7. Index Storage - Vector & Keyword
Embeddings and metadata are stored in Vespa (vector database). Vespa maintains both vector indexes for semantic search and keyword indexes for exact matching. Each document includes access control lists (ACLs) to enforce permissions at query time. Redis caches frequently accessed data.

### 8. User Query Processing
User submits natural language query through web interface or Slack bot. System applies query preprocessing: spell correction, expansion, and optional multi-language translation based on configured language hints.

### 9. Search Execution - Hybrid Retrieval
Query is converted to embedding vector (using same model as documents). Vespa executes hybrid search combining vector similarity (semantic) and keyword matching (BM25). Hybrid alpha parameter (0-1) controls weighting. Results are filtered by user's ACL permissions.

### 10. Re-ranking & Relevance Scoring
Top K documents (typically 20-50) undergo re-ranking using cross-encoder models that score query-document relevance more accurately than embedding similarity. Time decay factor boosts recent documents. Results are sorted by combined relevance score.

### 11. Context Preparation & Prompt Building
Top N ranked chunks (configurable, typically 5-10) are formatted into context. System applies prompt template specific to selected persona/assistant. Template includes system instructions, context chunks with metadata, user query, and any conversation history.

### 12. LLM Interaction - Answer Generation
Formatted prompt is sent to configured LLM provider (OpenAI, Anthropic, Azure OpenAI, local models via LiteLLM). System supports streaming responses for real-time user experience. LLM generates answer grounded in provided context, with citations to source documents.

### 13. Response Post-Processing
Answer is parsed to extract citations, format markdown, and validate content. System logs interaction for analytics: query text, documents retrieved, tokens used, response time. Citations link back to original source documents for verification.

### 14. Permission Validation & Document Access
When user clicks citation or requests document, system re-validates permissions against current ACL. If authorized, document content is retrieved from original source or cached version. Access attempts are logged for audit trails.

### 15. Background Maintenance - Index Updates
Celery workers continuously process indexing queue. New or updated documents flow through processing pipeline. Modified documents are re-chunked, re-embedded, and updated in Vespa. Deleted documents are removed from index. Status is tracked in PostgreSQL.

### 16. Background Maintenance - Permission Sync
Separate background jobs sync permissions from source systems. For each connector, system fetches current access control lists and updates document ACLs in Vespa. This ensures query-time permission filtering reflects current organizational structure.

### 17. Agent & Tool Execution
When chat involves custom agents, system identifies required tools (search, API calls, calculators). Agent framework orchestrates multi-step workflows, invoking tools sequentially or in parallel. Tool results are fed back to LLM for reasoning and response generation.

### 18. Feedback & Learning
Users can provide feedback (thumbs up/down) on answers. Feedback is stored with associated query, documents, and model parameters. System uses feedback to improve search ranking and identify knowledge gaps for administrators.

### 19. Analytics & Monitoring
System continuously logs metrics: query latency, document indexing rates, LLM token usage, error rates. Vespa timing information tracks search performance. PostgreSQL stores historical analytics. Telemetry data (optional) helps product improvement.

### 20. Administrative Operations
Admins manage system via web UI: configure LLM providers, create document sets (filtered collections), define personas/assistants with custom prompts, set rate limits, review usage analytics, and manage user groups and permissions.

---

## UiPath Component Mapping

The following table maps Onyx functionality to appropriate UiPath products:

| UiPath Product | Purpose |
|---------------|---------|
| **API Workflow** | Backend API Server - FastAPI application serving REST endpoints for authentication, connector management, search queries, chat interactions, and administrative operations. |
| **RPA Workflow** | Connector Data Ingestion Workflows - Automated workflows for each connector type (40+ connectors) that authenticate with external systems and fetch documents. |
| **RPA Workflow** | Document Processing Pipeline - Workflow that receives raw documents, performs content extraction, chunks documents intelligently, and outputs processed chunks. |
| **API Workflow** | Embedding Service Integration - API workflow that batches document chunks, makes HTTP requests to embedding model endpoint, and manages retry logic. |
| **Solution** | Index Management Solution - Coordinates interactions with Vespa vector database for storing/retrieving embeddings and executing hybrid searches. |
| **RPA Workflow** | Background Job Scheduler - Task queue implementation using UiPath Orchestrator queues for connector polling, document indexing, and maintenance tasks. |
| **Agent** | Conversational Chat Agent - AI agent that receives user queries, orchestrates search execution, formats prompts, and interacts with LLM APIs. |
| **Agent** | Custom Persona Agents - Specialized agents configured with specific prompts, knowledge domains, and capabilities for different use cases. |
| **Solution** | Authentication & Authorization Service - Manages user authentication flows, validates JWT tokens, maintains session state, and enforces RBAC policies. |
| **RPA Workflow** | Permission Synchronization Workflow - Periodically fetches access control lists from source systems and updates document ACLs. |
| **API Workflow** | Search & Retrieval Service - Handles search query execution by generating query embeddings, calling Vespa hybrid search, and applying re-ranking. |
| **Solution** | Database Management - PostgreSQL database operations for storing configurations, credentials, user accounts, chat history, and analytics. |
| **RPA Workflow** | LLM Provider Integration Workflows - Workflows for each LLM provider integration handling API authentication, request formatting, and streaming responses. |
| **Solution** | Cache Management - Redis-based caching layer for session data, documents, search results, and rate limit tracking. |
| **Solution** | Web Frontend Application - Next.js/React web application providing user interface for chat, search, and administration. |
| **Agent** | Slack Bot Agent - Autonomous agent that monitors Slack workspace, responds to mentions, executes searches, and formats responses. |
| **RPA Workflow** | Model Server Workflow - Hosts NLP models locally for embedding generation with GPU/CPU-accelerated inference. |
| **Solution** | Monitoring & Analytics Platform - Collects and aggregates system metrics, user behavior analytics, and cost tracking. |
| **API Workflow** | Webhook Handlers - Receive real-time notifications from external systems and trigger corresponding sync workflows. |
| **RPA Workflow** | Document Set Management - Automates creation and maintenance of document sets based on rules and filters. |

---

## Data Handling

### Inputs

The system processes the following types of input data:

- User credentials and authentication tokens (OAuth2, SAML assertions, API keys)
- Connector configuration parameters (API endpoints, workspace IDs, authentication credentials)
- Raw documents from 40+ sources: text files, PDFs, HTML pages, Office documents, emails, Slack messages, Jira tickets, etc.
- Natural language queries from users via web interface, Slack, or API
- User feedback on search results and chat responses (ratings, explicit feedback)
- Administrative configurations: LLM provider settings, embedding model selections, search parameters, rate limits
- Permission data from external systems: user groups, folder/channel access rights, organizational hierarchies
- Webhook notifications: real-time events from connected systems
- Conversation history and context from previous chat interactions
- Custom prompts and persona definitions from administrators

### Outputs

The system generates the following outputs:

- Processed document chunks with metadata stored in Vespa vector database
- Dense vector embeddings (768 or 1024 dimensions depending on model) for semantic search
- Hybrid search results with relevance scores, citations, and permission-filtered content
- AI-generated responses with citations to source documents, formatted in markdown
- Indexed documents with keyword inverted indexes for exact match searching
- Encrypted credentials stored in PostgreSQL with AES-256 encryption
- User session tokens (JWT) with configured expiration times
- Analytics data: query logs, document access patterns, token usage metrics, latency measurements
- Notification messages to users via Slack or email
- Administrative reports: connector status, indexing progress, error logs, usage statistics
- API responses in JSON format for programmatic access
- Audit logs for compliance: document access attempts, permission changes, configuration updates
- Exported document sets and curated knowledge collections
- Real-time streaming responses during chat interactions

### Transformations

Key data transformations include:

- **Raw document content → Cleaned text** via HTML parsing, PDF extraction, file format conversion
- **Long documents → Semantic chunks** (default 512 tokens) using intelligent splitting
- **Text chunks → Dense vector embeddings** (768/1024 dimensions) via transformer models
- **Natural language queries → Query embeddings** using same model as documents
- **External permission models → Normalized ACLs** compatible with Vespa filtering
- **Multi-source documents → Unified document schema** with standardized metadata
- **Search results → Re-ranked documents** using cross-encoder models
- **Retrieved context chunks → Formatted prompts** with persona-specific templates
- **LLM raw responses → Parsed answers** with extracted citations and formatted markdown
- **Connector-specific API responses → Standardized document objects**
- **User queries → Expanded queries** with synonyms, spell correction, and translation
- **Real-time webhook events → Queued background jobs**
- **Batch document updates → Delta operations in Vespa** for efficient maintenance
- **Historical conversations → Context windows with token limits**
- **Multiple LLM provider formats → Unified interface via LiteLLM proxy**
- **Raw feedback data → Aggregated analytics metrics**
- **System metrics and logs → Structured analytics data**
- **Time-series updates → Time decay scoring factors**

---

## Integration Points

### Google Workspace (Drive, Gmail, Calendar)
- **Method:** Google OAuth2 + Google APIs REST endpoints with client SDKs
- **Data Exchanged:**
  - Sent: OAuth tokens, API queries with file IDs, permission requests
  - Received: Document content, email messages, calendar events, user permissions, file metadata

### Slack Workspace
- **Method:** Slack OAuth2 + Events API (webhooks) + Web API
- **Data Exchanged:**
  - Sent: OAuth tokens, channel/message queries, chat message posts
  - Received: Channel messages, thread replies, file uploads, user profiles, workspace metadata

### Atlassian Confluence & Jira
- **Method:** OAuth 1.0a or Personal Access Tokens + Confluence/Jira REST APIs v2/v3
- **Data Exchanged:**
  - Sent: Auth tokens, CQL/JQL queries, page/issue IDs
  - Received: Wiki pages with attachments, issue tickets with comments, user permissions

### Microsoft 365 (SharePoint, Teams, OneDrive)
- **Method:** Microsoft OAuth2 (MSAL library) + Microsoft Graph API
- **Data Exchanged:**
  - Sent: Access tokens with specific scopes, GraphQL queries
  - Received: SharePoint site content, Teams channel messages, OneDrive files, user profiles

### Salesforce
- **Method:** OAuth2 JWT Bearer Flow + Salesforce REST/SOAP APIs
- **Data Exchanged:**
  - Sent: OAuth tokens, SOQL queries for objects
  - Received: CRM records with all fields, attachments, user permissions, object metadata

### GitHub
- **Method:** GitHub OAuth Apps or Personal Access Tokens + GitHub REST API v3
- **Data Exchanged:**
  - Sent: Auth tokens, repository queries, issue filters
  - Received: Repository files, issues, pull requests, wiki pages, commit history

### Zendesk
- **Method:** OAuth2 or API token authentication + Zendesk REST API
- **Data Exchanged:**
  - Sent: Auth credentials, ticket queries with time filters
  - Received: Support tickets with comments, knowledge base articles, user data

### LLM Providers (OpenAI, Anthropic, Azure OpenAI, AWS Bedrock)
- **Method:** HTTP REST APIs with API key authentication, streaming via Server-Sent Events
- **Data Exchanged:**
  - Sent: API keys, formatted prompts, temperature parameters, token limits
  - Received: Generated text completions, usage metrics, streaming response chunks

### Vespa Vector Database
- **Method:** HTTP REST API + Native Query Language (YQL)
- **Data Exchanged:**
  - Sent: Document JSONs with embeddings, YQL search queries with filters
  - Received: Search results with relevance scores, document retrievals, performance metrics

### PostgreSQL Database
- **Method:** Direct database connection via SQLAlchemy ORM
- **Data Exchanged:**
  - Sent: SQL queries for CRUD operations, transaction commits
  - Received: Connector configurations, user accounts, credentials, chat sessions, analytics

### Redis Cache
- **Method:** Direct Redis protocol connection with connection pooling
- **Data Exchanged:**
  - Sent: Cache set/get commands, session tokens, rate limit counters
  - Received: Cached data, distributed lock confirmations, pub/sub messages

### Embedding Model Service
- **Method:** HTTP REST API to internal Model Server or external services
- **Data Exchanged:**
  - Sent: Text chunks in batches, model name specification
  - Received: Dense vector embeddings, processing time metrics

### SMTP Email Server
- **Method:** SMTP protocol with TLS encryption
- **Data Exchanged:**
  - Sent: Authentication credentials, email messages with headers and body
  - Received: Delivery confirmations or bounce messages

### Identity Providers (Okta, Auth0, Azure AD)
- **Method:** SAML 2.0 or OIDC protocols, OAuth2 authorization code flow
- **Data Exchanged:**
  - Sent: SAML/OIDC authentication requests, redirect URIs
  - Received: SAML assertions or JWT ID tokens with user attributes

### Object Storage (AWS S3, Google Cloud Storage, R2, Azure Blob)
- **Method:** REST APIs with IAM authentication or access keys
- **Data Exchanged:**
  - Sent: Authentication credentials, bucket/object paths, file uploads
  - Received: File content downloads, metadata, directory listings

### Celery Task Queue
- **Method:** Message broker integration (Redis or RabbitMQ)
- **Data Exchanged:**
  - Sent: Serialized task definitions with arguments, task priorities
  - Received: Task execution confirmations, result data, error tracebacks

### Document360 Knowledge Base
- **Method:** REST API with API token authentication
- **Data Exchanged:**
  - Sent: API tokens, project version queries, article IDs
  - Received: Knowledge base articles with HTML content, categories, version information

### Notion Workspace
- **Method:** OAuth2 + Notion API
- **Data Exchanged:**
  - Sent: Integration tokens, page/database IDs, block queries
  - Received: Page content as blocks, database records with properties

### Gong Sales Intelligence
- **Method:** OAuth2 + Gong API
- **Data Exchanged:**
  - Sent: OAuth tokens, call queries with date filters
  - Received: Call transcripts, meeting metadata, recorded media references

### Linear Project Management
- **Method:** OAuth2 + Linear GraphQL API
- **Data Exchanged:**
  - Sent: API keys/OAuth tokens, GraphQL queries
  - Received: Issue details, project information, team data, workflow states

---

## Constraints and Assumptions

### Technical Requirements
1. **Python 3.11 runtime required** - UiPath must support or emulate Python execution environment
2. **Vespa vector database** assumed to be external service - UiPath must integrate via REST API
3. **GPU acceleration** required for embedding models at production scale
4. **Real-time streaming responses** from LLMs needed for good UX
5. **PostgreSQL required** for relational data storage
6. **Redis required** for caching and session management

### Architecture Assumptions
7. **OAuth2/SAML/OIDC** authentication flows assumed
8. **Celery task queue** for background jobs (can be replaced with UiPath Orchestrator queues)
9. **FastAPI** serves REST endpoints (can be replaced with UiPath API Workflows)
10. **Multi-tenancy support** with tenant-specific database schemas required
11. **Concurrent request handling** with async/await patterns
12. **JWT tokens** with configurable expiration for session management

### Security & Permissions
13. **Document ACLs** enforce permission filtering at query time
14. **Encrypted credentials** stored with AES-256 encryption
15. **Permission checks** before returning results to users
16. **Audit logs** for compliance tracking required

### Performance & Scale
17. **Search latency target** < 1 second
18. **Indexing throughput** target of 1000s documents per hour
19. **Vector embeddings** are 768 or 1024 dimensions
20. **Document chunks** default to 512 tokens
21. **LLM context windows** vary (4K to 200K+ tokens)
22. **Storage requirements** scale from 1GB to 100s TB

### Integration Constraints
23. **Connector rate limits** vary by source (Google 10K requests/day, etc.)
24. **Network connectivity** to external APIs required
25. **SSL/TLS certificates** required for production HTTPS
26. **Cost considerations** for LLM API usage ($0.001-0.06 per 1K tokens)

### Deployment & Operations
27. **Docker, Kubernetes, single-machine** deployment options
28. **High availability** assumes multiple replicas
29. **Database migrations** managed by Alembic (needs UiPath equivalent)
30. **Logging and monitoring** integration with external tools
31. **Backup and disaster recovery** procedures required
32. **Security scanning** and vulnerability management

### Functional Assumptions
33. **Web frontend** built with Next.js/React (may need custom UI in UiPath)
34. **Browser automation** for web connector uses Playwright (UiPath RPA can replace)
35. **Email verification** required for user registration in some deployments
36. **API rate limiting** to protect against abuse
37. **Token budgets** limit LLM usage per user/organization
38. **File uploads** limited to configurable size (default 10MB)
39. **Document retention** policies may require periodic cleanup

### Additional Considerations
40. **User onboarding** assumes technical competency for connector setup
41. **Model fine-tuning** not currently supported but on roadmap
42. **Query history and analytics** assume significant storage (100s GB)
43. **Telemetry collection** optional but enabled by default

---

## Open Questions

### Architecture & Technology Stack
1. How should UiPath handle the Python-based NLP models and embedding generation? Options: (1) Keep Python Model Server and call via API, (2) Translate to ONNX and use .NET runtime, (3) Use cloud-based embedding APIs exclusively.
2. Should the web frontend remain as-is (Next.js) and only backend be reimplemented in UiPath, or should UiPath Apps be used for UI?
3. How to handle vector database integration - is there a UiPath connector for Vespa, or must it be custom HTTP activities? Alternative: migrate to vector DB with better UiPath support?
4. Should LiteLLM proxy layer be kept as-is or reimplemented in UiPath for unified LLM provider interface?

### UiPath Product Usage
5. Can UiPath Agents support streaming responses from LLMs with acceptable latency? Or should streaming be handled by a lightweight API gateway?
6. How to map Celery's task priorities and ETAs to UiPath Orchestrator queue priorities and delayed execution?
7. Can UiPath handle the concurrent load of 100s-1000s of simultaneous chat requests, or is a traditional API gateway needed?
8. Should re-ranking models run in UiPath or external service? Re-ranking requires fast inference on 20-50 documents per query.

### Implementation Strategy
9. How to handle the 40+ connectors - implement all in UiPath, prioritize subset, or maintain hybrid with Python connectors behind API?
10. Should the Slack bot be an RPA workflow or remain as standalone service that calls UiPath API workflows?
11. Should the system remain as microservices architecture, or consolidate into monolithic UiPath solution for simpler deployment?
12. How to handle PDF parsing, OCR, and complex document processing - UiPath Document Understanding, or keep specialized Python libraries?

### Security & Operations
13. What is the strategy for managing secrets and credentials? UiPath Orchestrator Assets/Credential Store, or external vault like HashiCorp Vault?
14. How to implement multi-tenancy in UiPath - separate Orchestrator folders per tenant, or application-level tenant filtering?
15. How should database migrations be managed in UiPath - manual SQL scripts, dedicated .NET migration tool, or UiPath workflow?
16. How to implement distributed locking for concurrent access to shared resources in UiPath?

### Monitoring & Testing
17. What monitoring and alerting approach for UiPath workflows - Orchestrator Insights, Elasticsearch integration, or external APM tools?
18. What's the testing strategy for UiPath workflows - unit tests via UiPath Test Suite, or integration tests via external framework?
19. How to implement circuit breakers and retry policies for external API calls in UiPath - built-in retry scopes or custom logic?
20. Should query analytics and logging go to PostgreSQL or separate analytics database (ClickHouse, TimescaleDB)?

### Deployment & Scaling
21. What UiPath licensing model applies for the number of bots needed? Background job processing may require many concurrent unattended robots for indexing at scale.
22. What is the deployment topology - cloud Orchestrator vs on-premise, robot count and sizing, database server specifications?
23. How to optimize for cost - UiPath licensing costs vs current infrastructure costs (cloud compute, LLM APIs, vector DB)?
24. What's the disaster recovery plan - how quickly can UiPath workflows be redeployed, databases restored, and indexes rebuilt?

### Advanced Features
25. How to handle permission syncing - real-time via webhooks or scheduled polling? Impact on external system rate limits?
26. What's the approach for handling long-running indexing jobs - Job chaining in Orchestrator, or keep Celery for this component?
27. How to ensure exactly-once processing semantics for document updates when using UiPath queues?
28. How to handle schema evolution in Vespa indexes when document fields change - workflow to migrate existing documents?

### Process Improvements
29. How to handle upgrades and versioning of UiPath packages when multiple workflows are interdependent?
30. How to implement feature flags in UiPath for gradual rollout of new features?
31. What's the approach for handling time zones across global deployments?

---

## Next Steps

To move forward with UiPath implementation:

1. **Prioritize connectors** - Identify the 5-10 most critical data sources to implement first
2. **Proof of concept** - Build small prototype with one connector, basic search, and chat
3. **Architecture decisions** - Resolve open questions about technology stack and UiPath product usage
4. **Detailed design** - Create detailed workflow diagrams for each component
5. **Licensing assessment** - Evaluate UiPath licensing requirements based on expected scale
6. **Infrastructure planning** - Design deployment topology with robot counts and resource requirements
7. **Migration strategy** - Plan for gradual migration if replacing existing system
8. **Testing framework** - Establish testing approach for quality assurance
9. **Security review** - Validate security controls meet compliance requirements
10. **Performance benchmarking** - Test scalability and latency against requirements

---

## Document Information

- **Created:** 2025-02-19
- **Version:** 1.0
- **Source System:** Onyx (formerly Danswer) - https://github.com/onyx-dot-app/onyx
- **Target Platform:** UiPath Enterprise Automation Platform
- **Format:** Technical Specification for UiPath Implementation
