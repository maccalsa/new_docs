# Development Guide - Enhancing the Approved Premises UI

This guide explains how to enhance the Approved Premises application following the established patterns. The application uses a multi-step form system with a clear hierarchy and separation of concerns.

## Architecture Overview

The application follows a **Form → Section → Task → Page** hierarchy:

```
Form (e.g., Apply, Assess)
├── Section (e.g., ReasonsForPlacement)
│   ├── Task (e.g., BasicInformation)
│   │   ├── Page (e.g., SentenceType, Situation)
│   │   └── Page (e.g., ExamplePage)
│   └── Task (e.g., TypeOfAp)
└── Section (e.g., RiskAndNeedFactors)
```

### Key Components

- **Decorators**: `@Form`, `@Section`, `@Task`, `@Page` define the structure
- **Controllers**: Handle HTTP requests and form submission
- **Services**: Manage data and API calls
- **Nunjucks Templates**: Render the UI
- **TypeScript Classes**: Define page logic and validation

## Development Scenarios

### 1. Creating a New Page

To create a new page, you need to:

1. **Create the Page Class** (TypeScript)
2. **Add it to the Task** (update index.ts)
3. **Create the Template** (Nunjucks)
4. **Add Tests** (optional but recommended)

#### Example: Creating a Simple Yes/No Page

**Step 1: Create the Page Class**

```typescript
// server/form-pages/apply/reasons-for-placement/basic-information/myNewPage.ts
import type { TaskListErrors } from '@approved-premises/ui'
import { ApprovedPremisesApplication as Application } from '@approved-premises/api'
import { Page } from '../../../utils/decorators'
import TasklistPage from '../../../tasklistPage'

@Page({ name: 'my-new-page', bodyProperties: ['myChoice'] })
export default class MyNewPage implements TasklistPage {
  title = 'Do you want to proceed?'

  constructor(
    readonly body: { myChoice?: 'yes' | 'no' },
    readonly application: Application,
  ) {}

  response() {
    return { [this.title]: this.body.myChoice === 'yes' ? 'Yes' : 'No' }
  }

  previous() {
    return 'sentence-type' // Previous page in the flow
  }

  next() {
    return this.body.myChoice === 'yes' ? 'situation' : 'dashboard'
  }

  errors() {
    const errors: TaskListErrors<this> = {}

    if (!this.body.myChoice) {
      errors.myChoice = 'You must choose yes or no'
    }

    return errors
  }

  items() {
    return [
      {
        value: 'yes',
        text: 'Yes, I want to proceed',
        checked: this.body.myChoice === 'yes',
      },
      {
        value: 'no',
        text: 'No, I want to go back',
        checked: this.body.myChoice === 'no',
      },
    ]
  }
}
```

**Step 2: Add to Task**

```typescript
// server/form-pages/apply/reasons-for-placement/basic-information/index.ts
import MyNewPage from './myNewPage'

@Task({
  name: 'Basic Information',
  slug: 'basic-information',
  pages: [
    // ... existing pages
    MyNewPage, // Add your new page
  ],
})
export default class BasicInformation {}
```

**Step 3: Create Template**

```njk
<!-- server/views/applications/pages/basic-information/my-new-page.njk -->
{% extends "../layout.njk" %}

{% block questions %}
  {{
    formPageRadios(
      {
        fieldName: "myChoice",
        fieldset: {
          legend: {
            text: page.title,
            isPageHeading: true,
            classes: "govuk-fieldset__legend--l"
          }
        },
        items: page.items()
      },
      fetchContext()
    )
  }}
{% endblock %}
```

### 2. Creating a Form with Multiple Components

For complex forms with multiple components:

```typescript
// server/form-pages/apply/reasons-for-placement/basic-information/complexForm.ts
@Page({ name: 'complex-form', bodyProperties: ['name', 'email', 'phone', 'address'] })
export default class ComplexForm implements TasklistPage {
  title = 'Enter your details'

  constructor(
    readonly body: { 
      name?: string
      email?: string
      phone?: string
      address?: string
    },
    readonly application: Application,
  ) {}

  response() {
    return {
      'Full Name': this.body.name,
      'Email Address': this.body.email,
      'Phone Number': this.body.phone,
      'Address': this.body.address,
    }
  }

  previous() {
    return 'sentence-type'
  }

  next() {
    return 'situation'
  }

  errors() {
    const errors: TaskListErrors<this> = {}

    if (!this.body.name) {
      errors.name = 'You must enter your name'
    }
    if (!this.body.email) {
      errors.email = 'You must enter your email'
    }
    if (this.body.email && !this.isValidEmail(this.body.email)) {
      errors.email = 'Please enter a valid email address'
    }

    return errors
  }

  private isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
  }
}
```

**Template with Multiple Components:**

```njk
<!-- server/views/applications/pages/basic-information/complex-form.njk -->
{% extends "../layout.njk" %}

{% block questions %}
  <h1 class="govuk-heading-l">{{ page.title }}</h1>

  {{
    formPageInput({
      fieldName: "name",
      label: {
        text: "Full name",
        classes: "govuk-label--m"
      },
      classes: "govuk-input--width-20"
    }, fetchContext())
  }}

  {{
    formPageInput({
      fieldName: "email",
      label: {
        text: "Email address",
        classes: "govuk-label--m"
      },
      classes: "govuk-input--width-20",
      inputmode: 'email'
    }, fetchContext())
  }}

  {{
    formPageInput({
      fieldName: "phone",
      label: {
        text: "Phone number",
        classes: "govuk-label--m"
      },
      classes: "govuk-input--width-20",
      inputmode: 'tel'
    }, fetchContext())
  }}

  {{
    formPageTextarea({
      fieldName: "address",
      label: {
        text: "Address",
        classes: "govuk-label--m"
      }
    }, fetchContext())
  }}
{% endblock %}
```

### 3. Adding Form Components

The application provides several form components:

- `formPageInput` - Text inputs
- `formPageRadios` - Radio buttons
- `formPageCheckboxes` - Checkboxes
- `formPageSelect` - Dropdown selects
- `formPageTextarea` - Text areas
- `formPageDateInput` - Date inputs

**Example with Multiple Component Types:**

```njk
{% extends "../layout.njk" %}

{% block questions %}
  <h1 class="govuk-heading-l">{{ page.title }}</h1>

  <!-- Radio buttons -->
  {{
    formPageRadios({
      fieldName: "priority",
      fieldset: {
        legend: {
          text: "Priority level",
          classes: "govuk-fieldset__legend--m"
        }
      },
      items: [
        { value: "high", text: "High priority" },
        { value: "medium", text: "Medium priority" },
        { value: "low", text: "Low priority" }
      ]
    }, fetchContext())
  }}

  <!-- Checkboxes -->
  {{
    formPageCheckboxes({
      fieldName: "interests",
      fieldset: {
        legend: {
          text: "Areas of interest",
          classes: "govuk-fieldset__legend--m"
        }
      },
      items: [
        { value: "technology", text: "Technology" },
        { value: "sports", text: "Sports" },
        { value: "arts", text: "Arts" }
      ]
    }, fetchContext())
  }}

  <!-- Select dropdown -->
  {{
    formPageSelect({
      fieldName: "region",
      label: {
        text: "Select region",
        classes: "govuk-label--m"
      },
      items: [
        { value: "north", text: "North" },
        { value: "south", text: "South" },
        { value: "east", text: "East" },
        { value: "west", text: "West" }
      ]
    }, fetchContext())
  }}
{% endblock %}
```

### 4. Making Backend API Calls

To fetch data from the backend, you'll need to:

1. **Create a Service** (if it doesn't exist)
2. **Update the Controller** to use the service
3. **Pass data to the template**

**Example Service:**

```typescript
// server/services/myDataService.ts
import { RestClientBuilder } from '../data/restClientBuilder'

export default class MyDataService {
  constructor(private readonly clientFactory: RestClientBuilder<MyDataClient>) {}

  async getMyData(token: string): Promise<MyData[]> {
    const client = this.clientFactory(token)
    return client.getMyData()
  }

  async saveMyData(token: string, data: MyData): Promise<void> {
    const client = this.clientFactory(token)
    return client.saveMyData(data)
  }
}
```

**Update Controller:**

```typescript
// server/controllers/apply/applications/pagesController.ts
export default class PagesController {
  constructor(
    private readonly applicationService: ApplicationService,
    private readonly myDataService: MyDataService, // Add your service
    private readonly dataServices: Partial<DataServices>,
  ) {}

  show(taskName: TaskNames, pageName: string): RequestHandler {
    return async (req: Request, res: Response, next: NextFunction) => {
      try {
        const Page = getPage(taskName, pageName, 'applications')
        const { errors, errorSummary, userInput } = fetchErrorsAndUserInput(req)
        const page = await this.applicationService.initializePage(Page, req, this.dataServices, userInput)

        // Fetch additional data if needed
        let additionalData = {}
        if (pageName === 'my-new-page') {
          const myData = await this.myDataService.getMyData(req.user.token)
          additionalData = { myData }
        }

        res.render(viewPath(page, 'applications'), {
          applicationId: req.params.id,
          errors,
          errorSummary,
          task: taskName,
          page,
          ...page.body,
          ...additionalData, // Pass additional data to template
        })
      } catch (error) {
        // Error handling
      }
    }
  }
}
```

**Update Template to Use Data:**

```njk
{% extends "../layout.njk" %}

{% block questions %}
  <h1 class="govuk-heading-l">{{ page.title }}</h1>

  {% if myData %}
    <div class="govuk-inset-text">
      <p>Available options:</p>
      <ul class="govuk-list govuk-list--bullet">
        {% for item in myData %}
          <li>{{ item.name }} - {{ item.description }}</li>
        {% endfor %}
      </ul>
    </div>
  {% endif %}

  {{
    formPageSelect({
      fieldName: "selectedOption",
      label: {
        text: "Select an option",
        classes: "govuk-label--m"
      },
      items: myData.map(item => ({
        value: item.id,
        text: item.name
      }))
    }, fetchContext())
  }}
{% endblock %}
```

### 5. Data Validation

Validation happens in the page class's `errors()` method:

```typescript
@Page({ name: 'validation-example', bodyProperties: ['email', 'age', 'postcode'] })
export default class ValidationExample implements TasklistPage {
  title = 'Validation Example'

  constructor(
    readonly body: { 
      email?: string
      age?: string
      postcode?: string
    },
    readonly application: Application,
  ) {}

  errors() {
    const errors: TaskListErrors<this> = {}

    // Required field validation
    if (!this.body.email) {
      errors.email = 'Email is required'
    }

    // Email format validation
    if (this.body.email && !this.isValidEmail(this.body.email)) {
      errors.email = 'Please enter a valid email address'
    }

    // Age validation
    if (this.body.age) {
      const age = parseInt(this.body.age)
      if (isNaN(age) || age < 18 || age > 120) {
        errors.age = 'Age must be between 18 and 120'
      }
    }

    // Postcode validation (UK format)
    if (this.body.postcode && !this.isValidPostcode(this.body.postcode)) {
      errors.postcode = 'Please enter a valid UK postcode'
    }

    return errors
  }

  private isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
  }

  private isValidPostcode(postcode: string): boolean {
    return /^[A-Z]{1,2}[0-9][A-Z0-9]? ?[0-9][A-Z]{2}$/i.test(postcode)
  }

  // ... other methods
}
```

### 6. Submitting Data

Data submission is handled automatically by the framework:

1. **Form submits to controller** via `paths.applications.pages.update()`
2. **Controller calls service** to save data
3. **Service validates** and saves to API
4. **Redirects to next page** or shows errors

**Custom Submission Logic:**

```typescript
@Page({ 
  name: 'custom-submission', 
  bodyProperties: ['data'],
  controllerActions: { update: 'customUpdate' } // Custom controller action
})
export default class CustomSubmission implements TasklistPage {
  title = 'Custom Submission'

  constructor(
    readonly body: { data?: string },
    readonly application: Application,
  ) {}

  // ... other methods

  // Custom validation
  errors() {
    const errors: TaskListErrors<this> = {}

    if (!this.body.data) {
      errors.data = 'Data is required'
    }

    // Custom business logic validation
    if (this.body.data && this.body.data.length < 10) {
      errors.data = 'Data must be at least 10 characters'
    }

    return errors
  }
}
```

**Custom Controller Action:**

```typescript
// In your controller
customUpdate(taskName: TaskNames, pageName: string) {
  return async (req: Request, res: Response) => {
    const Page = getPage(taskName, pageName, 'applications')
    const page = await this.applicationService.initializePage(Page, req, this.dataServices)

    try {
      // Custom pre-save logic
      await this.customService.preProcess(page.body)
      
      // Save the page
      await this.applicationService.save(page, req)
      
      // Custom post-save logic
      await this.customService.postProcess(page.body)
      
      // Redirect
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
    } catch (error) {
      catchValidationErrorOrPropogate(
        req,
        res,
        error as Error,
        paths.applications.pages.show({ id: req.params.id, task: taskName, page: pageName }),
      )
    }
  }
}
```

### 7. Controlling Page Flow

#### 7a. Adding a New Page to Existing Flow

To add a page to an existing flow, update the `next()` method of the previous page:

```typescript
// In the previous page (e.g., sentenceType.ts)
next() {
  return 'my-new-page' // Instead of 'situation'
}
```

And update your new page's `previous()` and `next()` methods:

```typescript
// In your new page
previous() {
  return 'sentence-type'
}

next() {
  return 'situation' // Continue to the original next page
}
```

#### 7b. Breaking a Flow and Inserting a Page

To insert a page in the middle of an existing flow:

1. **Update the previous page** to point to your new page
2. **Update your new page** to point to the original next page
3. **Update the original next page** to point back to your new page as previous

```typescript
// Original flow: Page A → Page B → Page C

// Step 1: Update Page A
next() {
  return 'my-inserted-page' // Instead of 'page-b'
}

// Step 2: Your inserted page
previous() {
  return 'page-a'
}
next() {
  return 'page-b'
}

// Step 3: Update Page B
previous() {
  return 'my-inserted-page' // Instead of 'page-a'
}
```

**Conditional Flow Control:**

```typescript
@Page({ name: 'conditional-flow', bodyProperties: ['choice'] })
export default class ConditionalFlow implements TasklistPage {
  title = 'Make a choice'

  constructor(
    readonly body: { choice?: 'option1' | 'option2' | 'option3' },
    readonly application: Application,
  ) {}

  next() {
    switch (this.body.choice) {
      case 'option1':
        return 'page-for-option1'
      case 'option2':
        return 'page-for-option2'
      case 'option3':
        return 'page-for-option3'
      default:
        return 'default-page'
    }
  }

  // ... other methods
}
```

## Best Practices

1. **Follow the naming conventions**: Use kebab-case for page names, camelCase for properties
2. **Always implement all required methods**: `title`, `response`, `previous`, `next`, `errors`
3. **Use the existing form components**: Leverage `formPageInput`, `formPageRadios`, etc.
4. **Write tests**: Create test files for your pages
5. **Update the schema**: Run `npm run generate:schema:apply` after adding new pages
6. **Handle errors gracefully**: Always provide meaningful error messages
7. **Use TypeScript**: Leverage the type system for better code quality

## Common Patterns

- **Radio buttons**: Use `formPageRadios` with `items()` method
- **Text inputs**: Use `formPageInput` with validation
- **Conditional fields**: Use conditional HTML in radio/checkbox items
- **Data fetching**: Use services and pass data to templates
- **Validation**: Implement in the `errors()` method
- **Flow control**: Use the `next()` method for conditional routing

This guide should help you understand how to enhance the application following the established patterns. The key is to understand the hierarchy and use the existing infrastructure rather than building from scratch. 
