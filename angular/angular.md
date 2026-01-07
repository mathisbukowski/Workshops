# Angular Workshop - Building a Kanban Board

Welcome! In this workshop, you'll build a functional Kanban board while learning modern Angular concepts. Let's dive in.

## Setup

First, install Angular CLI globally and create your project:

```bash
npm install -g @angular/cli
ng new angular-kanban --standalone --routing --style=css
cd angular-kanban
ng serve
```

Your app should be running on http://localhost:4200

## How Angular Works

Angular is a framework for building dynamic web applications. Here's how it works briefly:

### Components are Building Blocks

Everything in Angular is a component. A component combines HTML (template), CSS (styles), and TypeScript (logic).

Example:

```typescript
import { Component } from "@angular/core";

@Component({
  selector: "app-welcome",
  template: `<h1>{{ title }}</h1>`,
  styles: [
    `
      h1 {
        color: blue;
      }
    `,
  ],
})
export class WelcomeComponent {
  title = "Welcome to Angular";
}
```

You use it in HTML like: `<app-welcome></app-welcome>`

### Data Binding

Angular keeps your template and data in sync:

```typescript
@Component({
  template: `
    <input [(ngModel)]="name" />
    <p>Hello {{ name }}!</p>
  `,
})
export class GreetingComponent {
  name = "World";
}
```

Type in the input, and the paragraph updates automatically.
The `ngModel` directive will bind the input's value to the component's name property, creating a two-way binding between the two.

### Event Handling

React to user actions:

```typescript
@Component({
  template: `
    <button (click)="sayHello()">Click me</button>
    <p>{{ message }}</p>
  `,
})
export class ButtonComponent {
  message = "";

  sayHello() {
    this.message = "Hello!";
  }
}
```

The `click` directive in Angular binds a method to the element’s click event, so the method is called whenever the element is clicked.

## Project Structure

Generate the components and services we'll need:

```bash
ng generate component components/kanban-board
ng generate component components/task-item
ng generate interface models/task
ng generate service store/task --skip-tests
```

## Key Concepts

### 1. Standalone Components

Standalone components are self-contained units that don't need modules. Everything they need is imported directly.

Example:

```typescript
@Component({
  selector: "app-greeting",
  standalone: true,
  template: `<h1>Hello {{ name }}!</h1>`,
})
export class GreetingComponent {
  name = "Angular";
}
```

### 2. Signals for State Management

Signals are Angular's reactive primitive for managing state. They automatically track dependencies and trigger updates.

Simple example:

```typescript
import { signal, computed } from "@angular/core";

export class CounterStore {
  count = signal(0);
  doubleCount = computed(() => this.count() * 2);

  increment() {
    this.count.update((value) => value + 1);
  }
}
```

This code creates a simple counter where the count value is stored and updated reactively, and doubleCount always shows twice the current count automatically.
Here’s how each works in Angular’s reactive state management:

- **signal**: Creates a special variable that automatically tracks its value and notifies the UI or other code when it changes. You read its value by calling it like a function (e.g., `count()`).

- **computed**: Defines a value that is automatically recalculated whenever any signals it depends on change. It’s like a formula that always stays up to date (e.g., `doubleCount` is always twice `count`).

- **update**: A method on a signal that lets you change its value based on the current value, often using a function (e.g., `count.update(value => value + 1)` increases `count` by 1).

More info here: https://angular.dev/guide/signals

### 3. Input/Output Communication

Components communicate through Input (parent to child) and Output (child to parent).

Example:

```typescript
// Child component
@Component({
  selector: 'app-button',
  template: `<button (click)="handleClick()">Click me</button>`
})
export class ButtonComponent {
  @Input() label = '';
  @Output() clicked = new EventEmitter<void>();

  handleClick() {
    this.clicked.emit();
  }
}

// Parent template
<app-button label="Save" (clicked)="onSave()"></app-button>
```

### 4. Control Flow Syntax

Angular uses @ syntax for conditionals and loops in templates.

Example:

```typescript
@Component({
  template: `
    @if (isLoggedIn) {
      <p>Welcome back!</p>
    } @else {
      <p>Please log in</p>
    }

    @for (item of items; track item.id) {
      <div>{{ item.name }}</div>
    }
  `
})
```

See more: https://angular.dev/guide/templates/control-flow

### 5. TypeScript Types and Interfaces

Define clear contracts for your data structures.

Example:

```typescript
export type Status = "pending" | "active" | "completed";

export interface Task {
  id: number;
  ...
}
```

## Building the Kanban

Before you start coding, here are the mistakes that most often slow beginners down, plus tiny examples (not full copy/paste solutions).

Your task model should include:

- id (number)
- title (string)
- status ('todo' | 'doing' | 'done')

Your TaskStore should:

- Store tasks in a signal
- Provide methods to add and move tasks between columns

Your components should:

- KanbanBoard: Display three columns, filter tasks by status
- TaskItem: Show a task with buttons to move it forward

### Common pitfalls (and how to avoid them)

- **Mixing up status strings**: if you use `'Todo'` in one place and `'todo'` elsewhere, filtering silently “breaks”. Prefer a union type and reuse it everywhere.

```ts
type TaskStatus = "todo" | "doing" | "done";
```

- **Forgetting that signals are functions**: reading a signal as `tasks` instead of `tasks()` won’t do what you expect.

```ts
const tasks = signal<Task[]>([]);
console.log(tasks());
```

- **Mutating arrays/objects in place**: `push`, `splice`, or editing an object property may not trigger updates unless you update the signal correctly.

```ts
tasks.update((list) => [...list, newTask]);
```

- **Filtering in the template instead of in code**: doing heavy logic inside the template is harder to debug and can get messy. Prefer `computed` values per column.

```ts
const todoTasks = computed(() => tasks().filter((t) => t.status === "todo"));
```

- **Forgetting to “track” in `@for`**: without a stable track key, DOM updates can look weird (items appear to jump, wrong button state, etc.).

```html
@for (t of todoTasks(); track t.id) {
<app-task-item [task]="t" />
}
```

- **Wrong move button conditions**: a tiny condition bug can show “move” buttons in the wrong columns.

  - If status is `'todo'`, show only “to doing”
  - If status is `'doing'`, show only “to done”
  - If status is `'done'`, show no move button

- **Output event type mismatch**: if the child emits a status but the parent expects a whole task (or the other way around), you’ll get confusing TypeScript/template errors.

```ts
@Output() move = new EventEmitter<TaskStatus>();
```

- **Using `[(ngModel)]` without importing forms**: `ngModel` won’t work unless forms support is imported in the component.

```ts
imports: [FormsModule];
```

- **Updating the wrong task**: a common bug is to update by array index (which changes after filtering). Prefer updating by `id`.

```ts
tasks.update((list) => list.map((t) => (t.id === id ? { ...t, status: "doing" } : t)));
```

## Challenge Tasks

Once the basic board works, try:

1. Add a form to create new tasks
2. Add delete functionality
3. Persist tasks to localStorage
4. Add drag and drop

## Debugging Tips

- Use Angular DevTools browser extension
- Check the console for errors
- Use `console.log()` to inspect signal values with `mySignal()`
- Remember: signals are functions, call them to get the value

## Resources

- Official docs: https://angular.dev

Good luck and have fun building!
