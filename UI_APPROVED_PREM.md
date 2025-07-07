# Approved Premises UI - Developer Guide

## Overview

The Approved Premises UI is a TypeScript/Node.js Express application that provides a web interface for managing approved premises applications and assessments. It uses server-side rendering with Nunjucks templates and integrates with multiple backend APIs through typed REST clients.

## Architecture

### Core Technologies
- **Runtime**: Node.js with Express.js
- **Language**: TypeScript
- **Templates**: Nunjucks with GOV.UK Design System
- **Authentication**: OAuth2 via HMPPS Auth
- **Session**: Redis-based session storage
- **Build**: ESBuild for bundling and development

### Application Structure
```
server/
├── app.ts                 # Express app setup
├── config.ts             # Configuration management
├── controllers/          # Request handlers
├── services/            # Business logic layer
├── data/               # API client layer
├── form-pages/         # Form system implementation
├── views/              # Nunjucks templates
├── middleware/         # Express middleware
└── utils/              # Utility functions
```

## Key Concepts

### 1. Form System Architecture

The application uses a sophisticated form system with a hierarchical structure:

```
Form → Sections → Tasks → Pages
```

Each level uses TypeScript decorators to define metadata and relationships:

```typescript
// Form level
@Form({ sections: [ReasonsForPlacement, RiskAndNeedFactors, MoveOn, AddDocuments, CheckYourAnswers] })
export default class Apply extends BaseForm {}

// Section level
@Section({ title: 'Type of AP required', tasks: [BasicInformation, TypeOfAp] })
export default class ReasonsForPlacement {}

// Task level
@Task({
  name: 'Basic Information',
  slug: 'basic-information',
  pages: [EnterRiskLevel, IsExceptionalCase, ConfirmYourDetails, ...],
})
export default class BasicInformation {}

// Page level
@Page({
  name: 'enter-risk-level',
  bodyProperties: ['riskLevel'],
})
export default class EnterRiskLevel implements TasklistPage {
  title = "We cannot check this person's tier"
  question = "What is the person's risk level?"
  
  // Navigation logic
  next() {
    if (this.body.riskLevel === 'highRisk' || this.body.riskLevel === 'veryHighRisk') {
      return 'is-exceptional-case'
    }
    return 'not-eligible'
  }
  
  // Validation
  errors() {
    const errors: TaskListErrors<this> = {}
    if (!this.body.riskLevel) {
      errors.riskLevel = 'You must state the risk level'
    }
    return errors
  }
  
  // Response formatting
  response() {
    return {
      [this.question]: riskLevels[this.body.riskLevel],
    }
  }
}
```

### 2. Data Flow

The application follows a layered architecture:

```
Client Request → Controller → Service → Data Access → External API
```

**Controllers** handle HTTP requests and responses:
```typescript
export default class PagesController {
  show(taskName: TaskNames, pageName: string): RequestHandler {
    return async (req: Request, res: Response, next: NextFunction) => {
      const Page = getPage(taskName, pageName, 'applications')
      const page = await this.applicationService.initializePage(Page, req, this.dataServices)
      
      res.render(viewPath(page, 'applications'), {
        applicationId: req.params.id,
        task: taskName,
        page,
        ...page.body,
      })
    }
  }
  
  update(taskName: TaskNames, pageName: string) {
    return async (req: Request, res: Response) => {
      const Page = getPage(taskName, pageName, 'applications')
      const page = await this.applicationService.initializePage(Page, req, this.dataServices)
      
      await this.applicationService.save(page, req)
      const next = page.next()
      
      if (next) {
        res.redirect(paths.applications.pages.show({ 
          id: req.params.id, 
          task: taskName, 
          page: page.next() 
        }))
      } else {
        res.redirect(paths.applications.show({ id: req.params.id }))
      }
    }
  }
}
```

**Services** contain business logic:
```typescript
export default class ApplicationService {
  async initializePage(
    Page: TasklistPageInterface,
    request: Request,
    dataServices: DataServices,
    userInput?: Record<string, unknown>,
  ): Promise<TasklistPage> {
    const application = await this.findApplication(request.user.token, request.params.id)
    const body = getBody(Page, application, request, userInput)
    
    const page = Page.initialize
      ? await Page.initialize(body, application, request.user.token, dataServices)
      : new Page(body, application)
    
    return page
  }
  
  async save(page: TasklistPage, request: Request) {
    const errors = page.errors()
    
    if (Object.keys(errors).length) {
      throw new ValidationError<typeof page>(errors)
    } else {
      const application = await this.findApplication(request.user.token, request.params.id)
      const updatedApplication = updateFormArtifactData(page, application, Review)
      
      const client = this.applicationClientFactory(request.user.token)
      await client.update(application.id, getApplicationUpdateData(updatedApplication))
    }
  }
}
```

**Data Access** provides typed API clients:
```typescript
export default class RestClient {
  async get({ path = '', query = '', headers = {}, responseType = '', raw = false }: GetRequest): Promise<unknown> {
    const result = await superagent
      .get(`${this.apiUrl()}${path}`)
      .agent(this.agent)
      .auth(this.token, { type: 'bearer' })
      .set({ ...this.defaultHeaders, ...headers })
      .timeout(this.timeoutConfig())
    
    return raw ? result : result.body
  }
}
```

### 3. Authentication & Middleware

The application uses a comprehensive middleware stack:

```typescript
// app.ts - Middleware setup order
app.use(setUpSentryRequestHandler(app))
app.use(methodOverride('_method'))
app.use(appInsightsMiddleware())
app.use(metricsMiddleware)
app.use(setUpHealthChecks())
app.use(setUpWebSecurity())
app.use(setUpWebSession())
app.use(setUpWebRequestParsing())
app.use(setUpStaticResources())
app.use(flash())
app.use(setUpAuthentication())
app.use(authorisationMiddleware())
app.use(setUpCsrf())
app.use(setUpCurrentUser(services))
```

**Authentication Flow**:
1. OAuth2 authentication via HMPPS Auth service
2. Session-based user management with Redis
3. Token-based API access for backend services
4. CSRF protection for form submissions

### 4. View System

Templates use Nunjucks with GOV.UK Design System components:

```njk
{# Layout template #}
{% extends "../../partials/layout.njk" %}

{% block content %}
  <div class="govuk-grid-row">
    <div class="govuk-grid-column-two-thirds">
      <form action="{{ paths.applications.pages.update({ id: applicationId, task: task, page: page.name }) }}?_method=PUT" method="post">
        <input type="hidden" name="_csrf" value="{{ csrfToken }}"/>
        
        {{ showErrorSummary(errorSummary) }}
        
        {% block questions %}{% endblock %}
        
        {{ govukButton({
            text: "Save and continue",
            preventDoubleClick: true
        }) }}
      </form>
    </div>
  </div>
{% endblock %}
```

**Form Components**:
```njk
{# Radio buttons component #}
{% macro formPageRadios(params, context) %}
  {% set radioItems = [] %}
  {% for item in params.items %}
    {% set radioItems = (radioItems.push(mergeObjects(item, {
      checked: context[params.fieldName] == item.value
    })), radioItems) %}
  {% endfor %}
  
  {{ govukRadios(
    mergeObjects(
      mergeObjects(params, { items: radioItems }),
      { idPrefix: params.fieldName, name: params.fieldName, errorMessage: context.errors[params.fieldName] }
    )
  ) }}
{% endmacro %}
```

## How to Create a New Screen

### 1. Create a New Page

```typescript
// server/form-pages/apply/your-section/your-task/yourPage.ts
import type { TaskListErrors } from '@approved-premises/ui'
import { Page } from '../../../utils/decorators'
import TasklistPage from '../../../tasklistPage'

type Body = {
  yourField?: string
}

@Page({
  name: 'your-page',
  bodyProperties: ['yourField'],
})
export default class YourPage implements TasklistPage {
  title = 'Your Page Title'
  question = 'Your question text?'
  
  yourField: string
  
  constructor(public body: Body) {
    this.yourField = body.yourField
  }
  
  previous() {
    return 'previous-page-name'
  }
  
  next() {
    if (this.body.yourField === 'some-value') {
      return 'next-page-name'
    }
    return 'alternative-page'
  }
  
  errors() {
    const errors: TaskListErrors<this> = {}
    
    if (!this.body.yourField) {
      errors.yourField = 'This field is required'
    }
    
    return errors
  }
  
  response() {
    return {
      [this.question]: this.body.yourField,
    }
  }
}
```

### 2. Add Page to Task

```typescript
// server/form-pages/apply/your-section/your-task/index.ts
import { Task } from '../../../utils/decorators'
import YourPage from './yourPage'
import OtherPage from './otherPage'

@Task({
  name: 'Your Task Name',
  slug: 'your-task',
  pages: [YourPage, OtherPage],
})
export default class YourTask {}
```

### 3. Add Task to Section

```typescript
// server/form-pages/apply/your-section/index.ts
import { Section } from '../../utils/decorators'
import YourTask from './your-task'
import OtherTask from './other-task'

@Section({ title: 'Your Section Title', tasks: [YourTask, OtherTask] })
export default class YourSection {}
```

### 4. Add Section to Form

```typescript
// server/form-pages/apply/index.ts
import YourSection from './your-section'
import { Form } from '../utils/decorators'
import BaseForm from '../baseForm'

@Form({ sections: [YourSection, ...otherSections] })
export default class Apply extends BaseForm {}
```

### 5. Create View Template

```njk
{# server/views/applications/pages/your-page.njk #}
{% extends "applications/pages/layout.njk" %}

{% block questions %}
  <h1 class="govuk-heading-l">{{ page.title }}</h1>
  
  {{ formPageRadios({
    fieldName: 'yourField',
    items: [
      { value: 'option1', text: 'Option 1' },
      { value: 'option2', text: 'Option 2' }
    ]
  }, context) }}
{% endblock %}
```

### 6. Add Routes

```typescript
// server/routes/apply.ts
export default function applyRoutes(controllers: Controllers, router: Router, services: Services) {
  // Routes are automatically generated based on form structure
  // No manual route addition needed
}
```

## How to Modify an Existing Screen

### 1. Modify Page Logic

```typescript
// Edit the existing page class
export default class ExistingPage implements TasklistPage {
  // Modify title, questions, validation logic, navigation logic
  title = 'Updated Title'
  
  next() {
    // Update navigation logic
    if (this.body.someField === 'new-value') {
      return 'new-next-page'
    }
    return 'existing-next-page'
  }
  
  errors() {
    const errors: TaskListErrors<this> = {}
    
    // Add new validation rules
    if (this.body.newField && !this.body.newField.trim()) {
      errors.newField = 'New field cannot be empty'
    }
    
    return errors
  }
}
```

### 2. Update View Template

```njk
{# Modify the existing template #}
{% block questions %}
  <h1 class="govuk-heading-l">{{ page.title }}</h1>
  
  {# Add new form fields #}
  {{ formPageInput({
    fieldName: 'newField',
    label: { text: 'New Field Label' }
  }, context) }}
  
  {# Modify existing fields #}
  {{ formPageRadios({
    fieldName: 'existingField',
    items: [
      { value: 'new-option', text: 'New Option' },
      { value: 'existing-option', text: 'Existing Option' }
    ]
  }, context) }}
{% endblock %}
```

### 3. Update Page Decorator

```typescript
@Page({
  name: 'existing-page',
  bodyProperties: ['existingField', 'newField'], // Add new properties
})
export default class ExistingPage implements TasklistPage {
  // Update constructor to handle new fields
  constructor(public body: Body) {
    this.existingField = body.existingField
    this.newField = body.newField // Add new field
  }
}
```

## Client-Server Flow

### 1. Request Flow

```
1. User navigates to /applications/{id}/pages/{task}/{page}
2. Express router matches route to controller
3. Controller calls service to initialize page
4. Service fetches application data from API
5. Page class is instantiated with form data
6. Controller renders template with page data
7. HTML is sent to browser
```

### 2. Form Submission Flow

```
1. User submits form via POST/PUT
2. Controller receives request with form data
3. Page class is instantiated with submitted data
4. Validation is performed via page.errors()
5. If valid, service saves data to API
6. Controller redirects to next page or dashboard
7. If invalid, page is re-rendered with errors
```

### 3. API Integration

```typescript
// Data flows through typed clients
const applicationClient = this.applicationClientFactory(token)
const application = await applicationClient.find(id)

// Form data is updated and saved
const updatedApplication = updateFormArtifactData(page, application, Review)
await client.update(application.id, getApplicationUpdateData(updatedApplication))
```

### 4. Session Management

```typescript
// User session contains authentication data
req.user.token // API access token
req.session // Session data stored in Redis

// CSRF protection
<input type="hidden" name="_csrf" value="{{ csrfToken }}"/>
```

## Development Workflow

### 1. Setup

```bash
# Install dependencies
npm install

# Generate environment files
./script/generate-dotenv-files.sh

# Start development server
npm run start:dev
```

### 2. Testing

```bash
# Unit tests
npm test

# Integration tests
npm run test:integration

# E2E tests
npm run test:e2e:local
```

### 3. Type Generation

```bash
# Update API types from OpenAPI schema
npm run generate-types
```

### 4. Schema Generation

```bash
# Generate form schemas
npm run generate:schema:apply
npm run generate:schema:assess
```

## Key Patterns to Remember

1. **Decorators define structure**: Use `@Form`, `@Section`, `@Task`, `@Page` decorators to define form hierarchy
2. **Pages implement TasklistPage**: All pages must implement the TasklistPage interface with required methods
3. **Navigation via next()/previous()**: Page navigation is controlled by these methods
4. **Validation in errors()**: Form validation logic goes in the errors() method
5. **Service layer abstraction**: Business logic belongs in services, not controllers
6. **Typed API clients**: All external API calls go through typed client classes
7. **GOV.UK Design System**: Use GOV.UK components for consistent UI
8. **CSRF protection**: All forms must include CSRF tokens
9. **Session-based auth**: User authentication is handled via OAuth2 and sessions

## Common Gotchas

1. **Page names must match**: The `name` in `@Page` decorator must match the filename
2. **Body properties must be declared**: All form fields must be listed in `bodyProperties`
3. **Navigation loops**: Ensure `next()` and `previous()` methods don't create infinite loops
4. **Type safety**: Use TypeScript types from `@approved-premises/api` for API data
5. **Error handling**: Always handle validation errors and API errors appropriately
6. **CSRF tokens**: Don't forget to include CSRF tokens in forms
7. **Session cleanup**: Ensure proper session cleanup on logout

This guide should provide a solid foundation for working with the Approved Premises UI application. The form system is the most complex part, but once you understand the decorator pattern and page lifecycle, it becomes much more manageable. 