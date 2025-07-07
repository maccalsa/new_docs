# Decorator Pattern & UI Flow Example

## Overview

This document explains how the decorator pattern works in the Approved Premises UI by walking through a concrete example: creating a "Submit Post" feature. We'll trace the complete flow from user interaction to API calls.

## The Decorator Pattern Explained

The application uses TypeScript decorators to define a hierarchical form structure:

```
@Form → @Section → @Task → @Page
```

Each decorator adds metadata that the framework uses to:
- Generate routes automatically
- Build navigation between pages
- Validate form data
- Render the correct templates

## Example: "Submit Post" Feature

Let's create a simple "Submit Post" feature to demonstrate the flow:

### 1. Page Definition

```typescript
// server/form-pages/posts/create-post/post-details.ts
import type { TaskListErrors } from '@approved-premises/ui'
import { Page } from '../../../utils/decorators'
import TasklistPage from '../../../tasklistPage'

type Body = {
  title?: string
  content?: string
  category?: 'news' | 'announcement' | 'update'
}

@Page({
  name: 'post-details',
  bodyProperties: ['title', 'content', 'category'],
})
export default class PostDetails implements TasklistPage {
  title = 'Create a new post'
  question = 'What are the details of your post?'
  
  title: string
  content: string
  category: string
  
  constructor(public body: Body) {
    this.title = body.title
    this.content = body.content
    this.category = body.category
  }
  
  previous() {
    return 'dashboard' // Go back to main dashboard
  }
  
  next() {
    if (this.body.title && this.body.content && this.body.category) {
      return 'review-post' // All fields filled, go to review
    }
    return '' // Stay on this page if validation fails
  }
  
  errors() {
    const errors: TaskListErrors<this> = {}
    
    if (!this.body.title?.trim()) {
      errors.title = 'You must enter a title'
    }
    
    if (!this.body.content?.trim()) {
      errors.content = 'You must enter content'
    }
    
    if (!this.body.category) {
      errors.category = 'You must select a category'
    }
    
    return errors
  }
  
  response() {
    return {
      'Post title': this.body.title,
      'Content': this.body.content,
      'Category': this.body.category,
    }
  }
}
```

### 2. Review Page

```typescript
// server/form-pages/posts/create-post/review-post.ts
import type { TaskListErrors } from '@approved-premises/ui'
import { Page } from '../../../utils/decorators'
import TasklistPage from '../../../tasklistPage'

type Body = {
  confirm?: 'yes' | 'no'
}

@Page({
  name: 'review-post',
  bodyProperties: ['confirm'],
})
export default class ReviewPost implements TasklistPage {
  title = 'Review your post'
  question = 'Are you ready to submit this post?'
  
  confirm: string
  
  constructor(public body: Body) {
    this.confirm = body.confirm
  }
  
  previous() {
    return 'post-details' // Go back to edit details
  }
  
  next() {
    if (this.body.confirm === 'yes') {
      return '' // Submit the form, no next page
    }
    return 'post-details' // Go back to edit if not confirmed
  }
  
  errors() {
    const errors: TaskListErrors<this> = {}
    
    if (!this.body.confirm) {
      errors.confirm = 'You must confirm before submitting'
    }
    
    return errors
  }
  
  response() {
    return {
      'Ready to submit': this.body.confirm === 'yes' ? 'Yes' : 'No',
    }
  }
}
```

### 3. Task Definition

```typescript
// server/form-pages/posts/create-post/index.ts
import { Task } from '../../../utils/decorators'
import PostDetails from './post-details'
import ReviewPost from './review-post'

@Task({
  name: 'Create Post',
  slug: 'create-post',
  pages: [PostDetails, ReviewPost],
})
export default class CreatePost {}
```

### 4. Section Definition

```typescript
// server/form-pages/posts/index.ts
import { Section } from '../../utils/decorators'
import CreatePost from './create-post'

@Section({ title: 'Post Management', tasks: [CreatePost] })
export default class Posts {}
```

### 5. Form Definition

```typescript
// server/form-pages/posts/form.ts
import { Form } from '../../utils/decorators'
import BaseForm from '../../baseForm'
import Posts from './index'

@Form({ sections: [Posts] })
export default class PostsForm extends BaseForm {}
```

## The Complete UI Flow

### Step 1: User Navigation

**User clicks**: "Create Post" button on dashboard

**URL**: `/posts/create-post/post-details`

**What happens**:
1. Express router matches the URL pattern
2. Routes are automatically generated from decorator metadata
3. Controller receives the request

### Step 2: Page Initialization

```typescript
// Controller receives request
const Page = getPage('create-post', 'post-details', 'posts')
const page = await this.postService.initializePage(Page, req, this.dataServices)

// Service initializes the page
async initializePage(Page: TasklistPageInterface, request: Request, dataServices: DataServices) {
  // FAKE API CALL #1: Get existing post data
  const post = await this.findPost(request.user.token, request.params.id)
  
  // Extract form data from request/API
  const body = getBody(Page, post, request, userInput)
  
  // Create page instance
  const page = new Page(body, post)
  
  return page
}
```

### Step 3: Template Rendering

```typescript
// Controller renders template
res.render('posts/pages/post-details', {
  postId: req.params.id,
  task: 'create-post',
  page,
  ...page.body, // title, content, category
})
```

**Template** (`server/views/posts/pages/post-details.njk`):
```njk
{% extends "posts/pages/layout.njk" %}

{% block questions %}
  <h1 class="govuk-heading-l">{{ page.title }}</h1>
  
  {{ formPageInput({
    fieldName: 'title',
    label: { text: 'Post title' }
  }, context) }}
  
  {{ formPageTextarea({
    fieldName: 'content',
    label: { text: 'Post content' }
  }, context) }}
  
  {{ formPageRadios({
    fieldName: 'category',
    items: [
      { value: 'news', text: 'News' },
      { value: 'announcement', text: 'Announcement' },
      { value: 'update', text: 'Update' }
    ]
  }, context) }}
{% endblock %}
```

### Step 4: User Submits Form

**User fills out form and clicks "Save and continue"**

**Form action**: `POST /posts/create-post/post-details?_method=PUT`

**What happens**:
1. Form data is sent to controller
2. Page class is instantiated with form data
3. Validation runs

```typescript
// Controller update method
update(taskName: TaskNames, pageName: string) {
  return async (req: Request, res: Response) => {
    const Page = getPage(taskName, pageName, 'posts')
    const page = await this.postService.initializePage(Page, req, this.dataServices)
    
    // Validation happens here
    await this.postService.save(page, req)
    
    const next = page.next() // Returns 'review-post'
    
    if (next) {
      // Redirect to next page
      res.redirect(paths.posts.pages.show({ 
        id: req.params.id, 
        task: taskName, 
        page: next 
      }))
    } else {
      // Form complete, go to dashboard
      res.redirect(paths.posts.show({ id: req.params.id }))
    }
  }
}
```

### Step 5: Service Validation & Save

```typescript
// Service save method
async save(page: TasklistPage, request: Request) {
  // Run validation
  const errors = page.errors()
  
  if (Object.keys(errors).length) {
    // Validation failed - throw error
    throw new ValidationError<typeof page>(errors)
  } else {
    // Validation passed - save data
    
    // FAKE API CALL #2: Get current post data
    const post = await this.findPost(request.user.token, request.params.id)
    
    // Update form data
    const updatedPost = updateFormArtifactData(page, post, Review)
    
    // FAKE API CALL #3: Save updated post
    const client = this.postClientFactory(request.user.token)
    await client.update(post.id, getPostUpdateData(updatedPost))
  }
}
```

### Step 6: Navigation to Review Page

**User is redirected to**: `/posts/create-post/review-post`

**Review page template**:
```njk
{% block questions %}
  <h1 class="govuk-heading-l">{{ page.title }}</h1>
  
  {# Show summary of previous page data #}
  {{ govukSummaryList({
    rows: [
      { key: { text: 'Title' }, value: { text: post.title } },
      { key: { text: 'Content' }, value: { text: post.content } },
      { key: { text: 'Category' }, value: { text: post.category } }
    ]
  }) }}
  
  {{ formPageRadios({
    fieldName: 'confirm',
    items: [
      { value: 'yes', text: 'Yes, submit this post' },
      { value: 'no', text: 'No, I want to make changes' }
    ]
  }, context) }}
{% endblock %}
```

### Step 7: Final Submission

**User confirms and clicks "Submit"**

**What happens**:
1. `page.next()` returns `''` (empty string) - no next page
2. Controller redirects to dashboard
3. Service saves final data

```typescript
// Final save with submission
async submit(page: TasklistPage, request: Request) {
  // FAKE API CALL #4: Submit the post
  const client = this.postClientFactory(request.user.token)
  await client.submit(post.id, getPostSubmissionData(post))
  
  // FAKE API CALL #5: Send notification
  await this.notificationService.sendPostNotification(post)
}
```

## How Decorators Generate Routes

The framework automatically generates routes based on decorator metadata:

```typescript
// Generated routes (you don't write these manually)
router.get('/posts/:id/pages/:task/:page', pagesController.show('create-post', 'post-details'))
router.put('/posts/:id/pages/:task/:page', pagesController.update('create-post', 'post-details'))
router.get('/posts/:id/pages/:task/:page', pagesController.show('create-post', 'review-post'))
router.put('/posts/:id/pages/:task/:page', pagesController.update('create-post', 'review-post'))
```

## Key Points About the Flow

### 1. **Decorators Define Structure**
- `@Form` defines the overall form
- `@Section` groups related tasks
- `@Task` groups related pages
- `@Page` defines individual form pages

### 2. **Metadata Drives Everything**
- Routes are auto-generated from page names
- Navigation is controlled by `next()` and `previous()` methods
- Validation is handled by `errors()` method
- Templates are matched by page name

### 3. **Service Layer Handles Business Logic**
- API calls happen in services, not controllers
- Validation errors are thrown and caught by controllers
- Data transformation happens in services

### 4. **Controllers Are Thin**
- Controllers only handle HTTP concerns
- They delegate to services for business logic
- They handle redirects and error responses

### 5. **Templates Are Data-Driven**
- Templates receive page data from controllers
- Form components are reusable
- Error display is handled automatically

## Common Patterns

### Navigation Logic
```typescript
next() {
  // Conditional navigation based on form data
  if (this.body.someField === 'value1') {
    return 'page-a'
  } else if (this.body.someField === 'value2') {
    return 'page-b'
  }
  return '' // End of form
}
```

### Validation
```typescript
errors() {
  const errors: TaskListErrors<this> = {}
  
  // Required field validation
  if (!this.body.requiredField) {
    errors.requiredField = 'This field is required'
  }
  
  // Conditional validation
  if (this.body.showExtraField && !this.body.extraField) {
    errors.extraField = 'Extra field is required when shown'
  }
  
  return errors
}
```

### API Integration
```typescript
// Service method
async save(page: TasklistPage, request: Request) {
  const errors = page.errors()
  
  if (Object.keys(errors).length) {
    throw new ValidationError<typeof page>(errors)
  }
  
  // Get current data
  const data = await this.getData(request.user.token, request.params.id)
  
  // Update with form data
  const updatedData = updateFormArtifactData(page, data, Review)
  
  // Save to API
  const client = this.clientFactory(request.user.token)
  await client.update(data.id, getUpdateData(updatedData))
}
```

This pattern ensures that:
- Form logic is centralized in page classes
- Navigation is predictable and testable
- API calls are properly abstracted
- Validation is consistent across the application
- Templates are simple and focused on presentation 