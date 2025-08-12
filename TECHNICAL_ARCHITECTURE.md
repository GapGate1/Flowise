# Flowise Technical Architecture Documentation

## Code Structure Analysis

### Node-Based Architecture

Flowise uses a node-based architecture where each component is a self-contained node that can be connected to form workflows. Each node implements the `INode` interface:

```typescript
interface INode {
    label: string        // Display name in UI
    name: string         // Unique identifier
    version: number      // Node version
    type: string         // Node type classification
    icon: string         // Icon filename
    category: string     // Category for grouping
    description: string  // Help text
    baseClasses: string[] // Inheritance chain
    inputs: INodeParams[] // Configuration parameters
    // Optional methods for dynamic behavior
    loadMethod?: string
    init?(nodeData: INodeData): Promise<any>
}
```

### Component Categories

#### 1. LLM Nodes (Language Models)
```typescript
// Example: OpenAI LLM Node
class OpenAI_LLMs implements INode {
    constructor() {
        this.label = 'OpenAI'
        this.name = 'openAI'
        this.version = 4.0
        this.type = 'OpenAI'
        this.category = 'LLMs'
        this.baseClasses = [this.type, ...getBaseClasses(OpenAI)]
        // Configuration inputs for API keys, model selection, etc.
    }
}
```

**Available LLM Integrations:**
- OpenAI (GPT-3.5, GPT-4, GPT-4 Turbo)
- Azure OpenAI
- Google Vertex AI (Gemini, PaLM)
- AWS Bedrock (Claude, Titan)
- Cohere (Command, Generate)
- Hugging Face Inference
- Ollama (Local models)
- Together AI, Replicate

#### 2. Agent Nodes
```typescript
// Example: Conversational Agent
class ConversationalAgent_Agents implements INode {
    constructor() {
        this.label = 'Conversational Agent'
        this.name = 'conversationalAgent'
        this.type = 'AgentExecutor'
        this.category = 'Agents'
        // Defines inputs: tools, model, memory, etc.
    }
}
```

**Agent Types:**
- **ConversationalAgent**: Chat-based interactions
- **ReActAgent**: Reasoning and Acting pattern
- **ToolAgent**: Tool-focused workflows
- **AutoGPT**: Autonomous task execution
- **CSVAgent**: Data analysis specialist

#### 3. Tool Nodes
Extensive toolkit for external integrations:

```typescript
// Tools are categorized by functionality:
- Search: GoogleSearchAPI, BraveSearch, TavilyAPI
- Productivity: Gmail, GoogleDocs, MicrosoftOutlook
- Development: GitHub, Jira, CodeInterpreter
- Web: WebScraper, WebBrowser, RequestsGet/Post
- AI: OpenAPIToolkit, ChainTool, AgentAsTool
```

### Server Architecture

#### Express.js Application Structure
```typescript
// Main server initialization
const app = express()
const server = http.createServer(app)

// Core middleware stack:
app.use(cors(getCorsOptions()))
app.use(express.json({ limit: '50mb' }))
app.use(express.urlencoded({ extended: true, limit: '50mb' }))
app.use(cookieParser())
app.use(sanitizeMiddleware)
app.use(expressRequestLogger)

// API routing
app.use('/api/v1', flowiseApiV1Router)
```

#### Database Layer (TypeORM)
```typescript
// Entity structure:
@Entity()
export class ChatFlow {
    @PrimaryGeneratedColumn('uuid')
    id: string

    @Column()
    name: string
    
    @Column('text')
    flowData: string
    
    @CreateDateColumn()
    createdDate: Date
    
    @UpdateDateColumn() 
    updatedDate: Date
}
```

**Key Entities:**
- `ChatFlow`: Workflow definitions
- `ChatMessage`: Conversation history
- `Tool`: Custom tool configurations
- `Credential`: API keys and secrets
- `Variable`: Global variables

#### Pool Management
```typescript
// NodesPool: Manages loaded node instances
export class NodesPool {
    private componentNodes: ICommonObject = {}
    private componentCredentials: ICommonObject = {}
    
    async initialize() {
        // Load all node definitions from packages/components
        await this.loadNodes()
        await this.loadCredentials()
    }
}
```

### Frontend Architecture (React)

#### Component Structure
```jsx
// Main App component with routing
function App() {
    return (
        <Provider store={store}>
            <Router>
                <Routes>
                    <Route path="/canvas/:id" element={<CanvasPage />} />
                    <Route path="/chatflows" element={<ChatflowsPage />} />
                    <Route path="/tools" element={<ToolsPage />} />
                </Routes>
            </Router>
        </Provider>
    )
}
```

#### State Management (Redux Toolkit)
```typescript
// Canvas slice for workflow editor
export const canvasSlice = createSlice({
    name: 'canvas',
    initialState: {
        nodes: [],
        edges: [],
        isDirty: false
    },
    reducers: {
        setNodes: (state, action) => {
            state.nodes = action.payload
        },
        setEdges: (state, action) => {
            state.edges = action.payload
        }
    }
})
```

#### Flow Editor (React Flow)
```jsx
// Visual flow editor component
const CanvasNode = ({ data, selected }) => {
    return (
        <div className={`node ${selected ? 'selected' : ''}`}>
            <NodeHeader icon={data.icon} label={data.label} />
            <NodeInputs inputs={data.inputs} />
            <NodeOutputs outputs={data.outputs} />
        </div>
    )
}
```

### Workflow Execution Engine

#### Flow Processing Pipeline
```typescript
// Workflow execution flow:
1. Parse flow definition (JSON)
2. Validate node connections
3. Resolve dependencies and execution order
4. Initialize node instances with configurations
5. Execute nodes in topological order
6. Handle data flow between nodes
7. Return results or stream responses
```

#### Agent Execution
```typescript
// AgentExecutor handles agent workflows
export class AgentExecutor {
    async call(input: ChainValues): Promise<ChainValues> {
        // 1. Parse user input
        // 2. Plan agent actions using LLM
        // 3. Execute tools as needed
        // 4. Synthesize final response
        // 5. Update conversation memory
    }
}
```

### Security Implementation

#### Credential Management
```typescript
// Encrypted credential storage
export const encryptCredentialData = (data: string): string => {
    const key = getEncryptionKey()
    const cipher = crypto.createCipher('aes256', key)
    let encrypted = cipher.update(data, 'utf8', 'hex')
    encrypted += cipher.final('hex')
    return encrypted
}
```

#### API Security
```typescript
// Authentication middleware
export const verifyToken = async (req: Request, res: Response, next: NextFunction) => {
    try {
        const token = req.headers.authorization?.split(' ')[1]
        if (!token) throw new Error('No token provided')
        
        const decoded = jwt.verify(token, JWT_SECRET)
        req.user = decoded
        next()
    } catch (error) {
        res.status(401).json({ message: 'Unauthorized' })
    }
}
```

### Performance Optimizations

#### Caching Strategy
```typescript
// Multi-level caching
export class CachePool {
    private nodeCache = new NodeCache({ stdTTL: 300 }) // 5 min TTL
    private redisCache: Redis // Distributed cache
    
    async get(key: string): Promise<any> {
        // 1. Check in-memory cache
        let result = this.nodeCache.get(key)
        if (result) return result
        
        // 2. Check Redis cache
        result = await this.redisCache.get(key)
        if (result) {
            this.nodeCache.set(key, result)
            return JSON.parse(result)
        }
        
        return null
    }
}
```

#### Streaming Responses
```typescript
// Server-Sent Events for real-time updates
export class SSEStreamer {
    stream(response: Response, data: any) {
        response.write(`data: ${JSON.stringify(data)}\n\n`)
    }
    
    end(response: Response) {
        response.write('data: [DONE]\n\n')
        response.end()
    }
}
```

### API Design

#### RESTful Endpoints
```typescript
// Core API routes
POST /api/v1/chatflows          // Create workflow
GET  /api/v1/chatflows/:id      // Get workflow
PUT  /api/v1/chatflows/:id      // Update workflow
DELETE /api/v1/chatflows/:id    // Delete workflow

POST /api/v1/prediction/:id     // Execute workflow
GET  /api/v1/chatmessages/:id   // Get chat history
POST /api/v1/tools             // Create custom tool
```

#### WebSocket Integration
```typescript
// Real-time communication
io.on('connection', (socket) => {
    socket.on('start-workflow', async (data) => {
        const result = await executeWorkflow(data.flowId, data.input)
        socket.emit('workflow-complete', result)
    })
})
```

### Error Handling

#### Centralized Error Management
```typescript
// Global error handler
export const errorHandlerMiddleware = (
    error: Error,
    req: Request,
    res: Response,
    next: NextFunction
) => {
    logger.error('API Error:', error)
    
    if (error.name === 'ValidationError') {
        return res.status(400).json({ message: error.message })
    }
    
    if (error.name === 'UnauthorizedError') {
        return res.status(401).json({ message: 'Unauthorized' })
    }
    
    res.status(500).json({ message: 'Internal Server Error' })
}
```

### Testing Strategy

#### Unit Testing
```typescript
// Jest test example
describe('OpenAI Node', () => {
    it('should initialize with correct configuration', () => {
        const openAINode = new OpenAI_LLMs()
        expect(openAINode.label).toBe('OpenAI')
        expect(openAINode.category).toBe('LLMs')
    })
    
    it('should execute successfully with valid input', async () => {
        const result = await openAINode.init(mockNodeData)
        expect(result).toBeDefined()
    })
})
```

#### Integration Testing
```typescript
// API endpoint testing
describe('Chatflow API', () => {
    it('should create and execute workflow', async () => {
        const response = await request(app)
            .post('/api/v1/chatflows')
            .send(mockFlowData)
            .expect(201)
            
        const flowId = response.body.id
        
        const execResponse = await request(app)
            .post(`/api/v1/prediction/${flowId}`)
            .send({ question: 'Hello' })
            .expect(200)
            
        expect(execResponse.body.text).toBeDefined()
    })
})
```

### Build and Deployment

#### Monorepo Build Process
```bash
# Using Turbo for optimized builds
turbo run build --parallel
# Builds all packages:
# - packages/server -> TypeScript compilation
# - packages/ui -> Vite build
# - packages/components -> TypeScript compilation
# - api-documentation -> Swagger generation
```

#### Docker Configuration
```dockerfile
# Multi-stage build for production
FROM node:18-alpine AS base
RUN npm install -g pnpm

FROM base AS deps
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

FROM base AS builder
COPY . .
COPY --from=deps /app/node_modules ./node_modules
RUN pnpm build

FROM base AS runner
COPY --from=builder /app/packages/server/dist ./dist
COPY --from=builder /app/packages/server/bin ./bin
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

This technical architecture provides a robust foundation for building, extending, and maintaining AI workflow applications with Flowise.