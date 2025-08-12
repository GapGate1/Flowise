# دليل المطور لمشروع Flowise | Flowise Developer Guide

## دليل البدء السريع | Quick Start Guide

### متطلبات النظام | System Requirements
```bash
Node.js >= 18.15.0
PNPM >= 9.0.0
TypeScript >= 5.4.5
```

### إعداد بيئة التطوير | Development Environment Setup

```bash
# 1. استنساخ المشروع | Clone the repository
git clone https://github.com/FlowiseAI/Flowise.git
cd Flowise

# 2. تثبيت PNPM | Install PNPM
npm install -g pnpm

# 3. تثبيت التبعيات | Install dependencies
pnpm install

# 4. بناء المشروع | Build the project
pnpm build

# 5. تشغيل للتطوير | Run in development mode
pnpm dev
```

### هيكل المجلدات | Directory Structure
```
Flowise/
├── packages/
│   ├── server/              # خادم Node.js | Node.js server
│   │   ├── src/
│   │   │   ├── controllers/ # وحدات التحكم | Controllers
│   │   │   ├── routes/      # المسارات | Routes
│   │   │   ├── services/    # الخدمات | Services
│   │   │   └── database/    # قاعدة البيانات | Database
│   │   └── bin/             # ملفات تشغيل | Executable files
│   ├── ui/                  # واجهة React | React UI
│   │   ├── src/
│   │   │   ├── views/       # الصفحات | Pages
│   │   │   ├── components/  # المكونات | Components
│   │   │   └── store/       # إدارة الحالة | State management
│   ├── components/          # عقد النظام | System nodes
│   │   ├── nodes/           # تعريفات العقد | Node definitions
│   │   └── credentials/     # بيانات الاعتماد | Credentials
│   └── api-documentation/   # توثيق API | API documentation
```

## إنشاء عقدة مخصصة | Creating Custom Nodes

### 1. إنشاء عقدة LLM جديدة | Creating a New LLM Node

```typescript
// packages/components/nodes/llms/CustomLLM/CustomLLM.ts
import { INode, INodeData, INodeParams } from '../../../src/Interface'
import { getBaseClasses } from '../../../src/utils'

class CustomLLM_LLMs implements INode {
    label: string
    name: string
    version: number
    type: string
    icon: string
    category: string
    description: string
    baseClasses: string[]
    credential: INodeParams
    inputs: INodeParams[]

    constructor() {
        this.label = 'Custom LLM'
        this.name = 'customLLM'
        this.version = 1.0
        this.type = 'CustomLLM'
        this.icon = 'custom.svg'
        this.category = 'LLMs'
        this.description = 'Custom language model integration'
        this.baseClasses = [this.type]
        
        this.credential = {
            label: 'Connect Credential',
            name: 'credential',
            type: 'credential',
            credentialNames: ['customLLMApi']
        }
        
        this.inputs = [
            {
                label: 'Model Name',
                name: 'modelName',
                type: 'string',
                default: 'custom-model-v1'
            },
            {
                label: 'Temperature',
                name: 'temperature',
                type: 'number',
                step: 0.1,
                default: 0.7,
                optional: true
            },
            {
                label: 'Max Tokens',
                name: 'maxTokens',
                type: 'number',
                optional: true
            }
        ]
    }

    async init(nodeData: INodeData): Promise<CustomLLM> {
        const modelName = nodeData.inputs?.modelName as string
        const temperature = nodeData.inputs?.temperature as number
        const maxTokens = nodeData.inputs?.maxTokens as number
        const credentialData = await getCredentialData(nodeData.credential)
        
        const apiKey = credentialData.apiKey
        
        return new CustomLLM({
            modelName,
            temperature,
            maxTokens,
            apiKey
        })
    }
}

// تصدير العقدة | Export the node
module.exports = { nodeClass: CustomLLM_LLMs }
```

### 2. إنشاء أداة مخصصة | Creating a Custom Tool

```typescript
// packages/components/nodes/tools/CustomTool/CustomTool.ts
import { Tool } from '@langchain/core/tools'
import { INode, INodeData, INodeParams } from '../../../src/Interface'

class CustomTool_Tools implements INode {
    label: string
    name: string
    version: number
    description: string
    type: string
    icon: string
    category: string
    baseClasses: string[]
    inputs: INodeParams[]

    constructor() {
        this.label = 'Custom Tool'
        this.name = 'customTool'
        this.version = 1.0
        this.type = 'CustomTool'
        this.icon = 'tool.svg'
        this.category = 'Tools'
        this.description = 'Custom tool for specific functionality'
        this.baseClasses = [this.type, 'Tool']
        
        this.inputs = [
            {
                label: 'Tool Name',
                name: 'toolName',
                type: 'string',
                placeholder: 'my_custom_tool'
            },
            {
                label: 'Tool Description',
                name: 'toolDesc',
                type: 'string',
                rows: 3,
                placeholder: 'Description of what the tool does'
            },
            {
                label: 'API Endpoint',
                name: 'apiEndpoint',
                type: 'string',
                placeholder: 'https://api.example.com/endpoint'
            }
        ]
    }

    async init(nodeData: INodeData): Promise<Tool> {
        const toolName = nodeData.inputs?.toolName as string
        const toolDesc = nodeData.inputs?.toolDesc as string
        const apiEndpoint = nodeData.inputs?.apiEndpoint as string

        return new CustomAPITool(toolName, toolDesc, apiEndpoint)
    }
}

class CustomAPITool extends Tool {
    name: string
    description: string
    apiEndpoint: string

    constructor(name: string, description: string, apiEndpoint: string) {
        super()
        this.name = name
        this.description = description
        this.apiEndpoint = apiEndpoint
    }

    async _call(input: string): Promise<string> {
        try {
            const response = await fetch(this.apiEndpoint, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ query: input })
            })
            const data = await response.json()
            return JSON.stringify(data)
        } catch (error) {
            return `Error calling API: ${error.message}`
        }
    }
}

module.exports = { nodeClass: CustomTool_Tools }
```

### 3. إنشاء وكيل مخصص | Creating a Custom Agent

```typescript
// packages/components/nodes/agents/CustomAgent/CustomAgent.ts
import { AgentExecutor } from '../../../src/agents'
import { INode, INodeData, INodeParams } from '../../../src/Interface'

class CustomAgent_Agents implements INode {
    label: string
    name: string
    version: number
    description: string
    type: string
    icon: string
    category: string
    baseClasses: string[]
    inputs: INodeParams[]

    constructor() {
        this.label = 'Custom Agent'
        this.name = 'customAgent'
        this.version = 1.0
        this.type = 'AgentExecutor'
        this.category = 'Agents'
        this.icon = 'agent.svg'
        this.description = 'Custom agent with specialized behavior'
        this.baseClasses = [this.type]
        
        this.inputs = [
            {
                label: 'LLM Model',
                name: 'model',
                type: 'BaseChatModel'
            },
            {
                label: 'Tools',
                name: 'tools',
                type: 'Tool',
                list: true
            },
            {
                label: 'Agent Prompt',
                name: 'systemPrompt',
                type: 'string',
                rows: 4,
                default: 'You are a helpful AI assistant specialized in...'
            },
            {
                label: 'Max Iterations',
                name: 'maxIterations',
                type: 'number',
                default: 5,
                optional: true
            }
        ]
    }

    async init(nodeData: INodeData): Promise<AgentExecutor> {
        const model = nodeData.inputs?.model
        const tools = nodeData.inputs?.tools as Tool[]
        const systemPrompt = nodeData.inputs?.systemPrompt as string
        const maxIterations = nodeData.inputs?.maxIterations as number

        const agent = await createCustomAgent({
            llm: model,
            tools,
            systemPrompt
        })

        return new AgentExecutor({
            agent,
            tools,
            maxIterations,
            verbose: true
        })
    }
}

module.exports = { nodeClass: CustomAgent_Agents }
```

## تطوير واجهة المستخدم | Frontend Development

### إضافة صفحة جديدة | Adding a New Page

```jsx
// packages/ui/src/views/CustomPage/index.jsx
import { useState, useEffect } from 'react'
import { Grid, Card, CardContent, Typography, Button } from '@mui/material'
import MainCard from 'ui-component/cards/MainCard'

const CustomPage = () => {
    const [data, setData] = useState([])
    const [loading, setLoading] = useState(false)

    useEffect(() => {
        loadData()
    }, [])

    const loadData = async () => {
        setLoading(true)
        try {
            const response = await fetch('/api/v1/custom-data')
            const result = await response.json()
            setData(result)
        } catch (error) {
            console.error('Error loading data:', error)
        } finally {
            setLoading(false)
        }
    }

    return (
        <MainCard title="Custom Page">
            <Grid container spacing={3}>
                <Grid item xs={12}>
                    <Card>
                        <CardContent>
                            <Typography variant="h6" gutterBottom>
                                Custom Functionality
                            </Typography>
                            <Button 
                                variant="contained" 
                                onClick={loadData}
                                disabled={loading}
                            >
                                {loading ? 'Loading...' : 'Refresh Data'}
                            </Button>
                        </CardContent>
                    </Card>
                </Grid>
            </Grid>
        </MainCard>
    )
}

export default CustomPage
```

### إضافة مكون قابل للإعادة الاستخدام | Adding a Reusable Component

```jsx
// packages/ui/src/ui-component/CustomComponent.jsx
import { Card, CardContent, Typography, IconButton } from '@mui/material'
import { IconEdit, IconDelete } from '@tabler/icons-react'

const CustomComponent = ({ title, description, onEdit, onDelete }) => {
    return (
        <Card sx={{ mb: 2 }}>
            <CardContent>
                <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'start' }}>
                    <div>
                        <Typography variant="h6" gutterBottom>
                            {title}
                        </Typography>
                        <Typography variant="body2" color="text.secondary">
                            {description}
                        </Typography>
                    </div>
                    <div>
                        <IconButton onClick={onEdit} size="small">
                            <IconEdit />
                        </IconButton>
                        <IconButton onClick={onDelete} size="small" color="error">
                            <IconDelete />
                        </IconButton>
                    </div>
                </div>
            </CardContent>
        </Card>
    )
}

export default CustomComponent
```

## تطوير الخادم | Backend Development

### إضافة API جديد | Adding New API Endpoints

```typescript
// packages/server/src/routes/custom.ts
import express from 'express'
import { CustomController } from '../controllers/CustomController'

const router = express.Router()

// GET /api/v1/custom
router.get('/', CustomController.getAll)

// POST /api/v1/custom
router.post('/', CustomController.create)

// GET /api/v1/custom/:id
router.get('/:id', CustomController.getById)

// PUT /api/v1/custom/:id
router.put('/:id', CustomController.update)

// DELETE /api/v1/custom/:id
router.delete('/:id', CustomController.delete)

export default router
```

### إنشاء كونترولر | Creating a Controller

```typescript
// packages/server/src/controllers/CustomController.ts
import { Request, Response } from 'express'
import { CustomService } from '../services/CustomService'
import { handleError } from '../utils/errorHandler'

export class CustomController {
    static async getAll(req: Request, res: Response) {
        try {
            const items = await CustomService.findAll()
            res.json(items)
        } catch (error) {
            handleError(error, res)
        }
    }

    static async create(req: Request, res: Response) {
        try {
            const item = await CustomService.create(req.body)
            res.status(201).json(item)
        } catch (error) {
            handleError(error, res)
        }
    }

    static async getById(req: Request, res: Response) {
        try {
            const item = await CustomService.findById(req.params.id)
            if (!item) {
                return res.status(404).json({ message: 'Item not found' })
            }
            res.json(item)
        } catch (error) {
            handleError(error, res)
        }
    }

    static async update(req: Request, res: Response) {
        try {
            const item = await CustomService.update(req.params.id, req.body)
            res.json(item)
        } catch (error) {
            handleError(error, res)
        }
    }

    static async delete(req: Request, res: Response) {
        try {
            await CustomService.delete(req.params.id)
            res.status(204).send()
        } catch (error) {
            handleError(error, res)
        }
    }
}
```

### إنشاء خدمة | Creating a Service

```typescript
// packages/server/src/services/CustomService.ts
import { AppDataSource } from '../DataSource'
import { CustomEntity } from '../database/entities/CustomEntity'

export class CustomService {
    private static repository = AppDataSource.getRepository(CustomEntity)

    static async findAll(): Promise<CustomEntity[]> {
        return await this.repository.find()
    }

    static async findById(id: string): Promise<CustomEntity | null> {
        return await this.repository.findOne({ where: { id } })
    }

    static async create(data: Partial<CustomEntity>): Promise<CustomEntity> {
        const item = this.repository.create(data)
        return await this.repository.save(item)
    }

    static async update(id: string, data: Partial<CustomEntity>): Promise<CustomEntity> {
        await this.repository.update(id, data)
        const updated = await this.findById(id)
        if (!updated) throw new Error('Item not found')
        return updated
    }

    static async delete(id: string): Promise<void> {
        await this.repository.delete(id)
    }
}
```

### إنشاء كائن قاعدة البيانات | Creating Database Entity

```typescript
// packages/server/src/database/entities/CustomEntity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, UpdateDateColumn } from 'typeorm'

@Entity({ name: 'custom_items' })
export class CustomEntity {
    @PrimaryGeneratedColumn('uuid')
    id: string

    @Column()
    name: string

    @Column('text', { nullable: true })
    description: string

    @Column('simple-json', { nullable: true })
    metadata: Record<string, any>

    @CreateDateColumn()
    createdDate: Date

    @UpdateDateColumn()
    updatedDate: Date
}
```

## اختبار التطبيق | Testing

### اختبار الوحدة | Unit Testing

```typescript
// packages/server/test/CustomService.test.ts
import { CustomService } from '../src/services/CustomService'
import { AppDataSource } from '../src/DataSource'

describe('CustomService', () => {
    beforeAll(async () => {
        await AppDataSource.initialize()
    })

    afterAll(async () => {
        await AppDataSource.destroy()
    })

    beforeEach(async () => {
        // تنظيف البيانات | Clean up data
        await AppDataSource.getRepository('CustomEntity').clear()
    })

    it('should create a new item', async () => {
        const data = {
            name: 'Test Item',
            description: 'Test Description'
        }

        const item = await CustomService.create(data)
        expect(item.id).toBeDefined()
        expect(item.name).toBe(data.name)
    })

    it('should find all items', async () => {
        await CustomService.create({ name: 'Item 1' })
        await CustomService.create({ name: 'Item 2' })

        const items = await CustomService.findAll()
        expect(items).toHaveLength(2)
    })
})
```

### اختبار API | API Testing

```typescript
// packages/server/test/custom.api.test.ts
import request from 'supertest'
import { app } from '../src/index'

describe('Custom API', () => {
    it('should create and retrieve item', async () => {
        const data = {
            name: 'Test Item',
            description: 'Test Description'
        }

        // إنشاء عنصر | Create item
        const createResponse = await request(app)
            .post('/api/v1/custom')
            .send(data)
            .expect(201)

        expect(createResponse.body.id).toBeDefined()
        expect(createResponse.body.name).toBe(data.name)

        // استرجاع العنصر | Retrieve item
        const getResponse = await request(app)
            .get(`/api/v1/custom/${createResponse.body.id}`)
            .expect(200)

        expect(getResponse.body.name).toBe(data.name)
    })
})
```

## النشر | Deployment

### إعداد Docker | Docker Setup

```dockerfile
# Dockerfile.custom
FROM node:18-alpine

WORKDIR /app

# نسخ ملفات المشروع | Copy project files
COPY package*.json ./
COPY pnpm-lock.yaml ./

# تثبيت pnpm | Install pnpm
RUN npm install -g pnpm

# تثبيت التبعيات | Install dependencies
RUN pnpm install

# نسخ الكود المصدري | Copy source code
COPY . .

# بناء المشروع | Build project
RUN pnpm build

# تشغيل التطبيق | Run application
EXPOSE 3000
CMD ["pnpm", "start"]
```

### docker-compose للتطوير | Development docker-compose

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  flowise:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_TYPE=postgres
      - DATABASE_HOST=postgres
      - DATABASE_PORT=5432
      - DATABASE_USERNAME=flowise
      - DATABASE_PASSWORD=flowise
      - DATABASE_NAME=flowise
    volumes:
      - ./packages/server/src:/app/packages/server/src
      - ./packages/ui/src:/app/packages/ui/src
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_USER=flowise
      - POSTGRES_PASSWORD=flowise
      - POSTGRES_DB=flowise
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

## نصائح التطوير | Development Tips

### 1. التصحيح | Debugging

```typescript
// استخدام logger مدمج | Use built-in logger
import logger from '../utils/logger'

logger.info('Processing request', { userId: req.user?.id })
logger.error('Database error', error)
logger.debug('Query result', { result })
```

### 2. التحقق من صحة البيانات | Data Validation

```typescript
// استخدام joi للتحقق | Use joi for validation
import Joi from 'joi'

const schema = Joi.object({
    name: Joi.string().required().min(3).max(50),
    description: Joi.string().optional(),
    metadata: Joi.object().optional()
})

const { error, value } = schema.validate(req.body)
if (error) {
    return res.status(400).json({ message: error.details[0].message })
}
```

### 3. إدارة الأخطاء | Error Management

```typescript
// كلاس خطأ مخصص | Custom error class
export class CustomError extends Error {
    statusCode: number
    
    constructor(message: string, statusCode: number = 500) {
        super(message)
        this.statusCode = statusCode
        this.name = 'CustomError'
    }
}

// استخدام | Usage
throw new CustomError('Resource not found', 404)
```

### 4. التحسين | Optimization

```typescript
// استخدام التخزين المؤقت | Use caching
import { CachePool } from '../CachePool'

const cacheKey = `user:${userId}`
let userData = await CachePool.get(cacheKey)

if (!userData) {
    userData = await getUserFromDatabase(userId)
    await CachePool.set(cacheKey, userData, 300) // 5 minutes
}
```

هذا الدليل يوفر الأساسيات لتطوير وتوسيع مشروع Flowise بطريقة احترافية ومنظمة.

This guide provides the fundamentals for developing and extending the Flowise project in a professional and organized manner.