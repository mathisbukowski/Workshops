# NestJS Workshop: Building a Real-Time Collaborative Kanban API

**Topic:** Advanced GraphQL, Authentication, and Real-Time Data with PostgreSQL.
**Goal:** Build a secure, multi-user, real-time Kanban API with an optimized data layer.

---

## Core Resources
*   [NestJS GraphQL Docs](https://docs.nestjs.com/graphql/quick-start)
*   [NestJS Authentication Docs (Passport)](https://docs.nestjs.com/security/authentication)
*   [TypeORM Relations Docs](https://typeorm.io/relations)

---

## Phase 1: The Foundation - Setup & Config

**Objective:** Get a clean, configurable, and database-connected project running.

### Your Mission:
1.  **Project Setup:**
    *   Initialize a new NestJS project.
    *   Install all dependencies: `@nestjs/graphql`, `@nestjs/apollo`, `@nestjs/typeorm`, `@nestjs/config`, `graphql`, `@apollo/server`, `typeorm`, `pg`.
2.  **Docker & Postgres:**
    *   Create a `docker-compose.yml` for a PostgreSQL database.
    *   Ensure you can connect to it using a DB client (like DBeaver or TablePlus).
3.  **Configuration Management (No Hardcoding!):**
    *   Use the `@nestjs/config` module. Create a `.env` file for your database credentials (`DB_HOST`, `DB_USER`, etc.).
    *   Update `app.module.ts` to use `ConfigModule` and inject `ConfigService` to configure TypeORM. This is a professional practice.
4.  **GraphQL Module:**
    *   Configure the `GraphQLModule` with the Apollo driver.
    *   Set `autoSchemaFile` to `true` and enable the `playground`.

> **✅ Checkpoint:** The server starts without errors and you can access the GraphQL Playground at `http://localhost:3000/graphql`.

---

##  Phase 2: User Identity - Authentication with JWT

**Objective:** Secure the platform. We need users who can sign up and log in.

### Key Concepts:
*   **JWT (JSON Web Token):** A standard for creating access tokens.
*   **Passport.js:** The library NestJS uses for authentication strategies.
*   **Guards:** NestJS middleware that protects routes (or in our case, resolvers).

### Your Mission:
1.  **User Entity:** Create a `User` entity with `id`, `email`, and `password` fields.
    *   **Security:** The password should **never** be stored in plain text. Use a library like `bcrypt` to hash it before saving.
    *   **GraphQL:** Expose `id` and `email`, but **never** the password field in the GraphQL schema.
2.  **Authentication Module:** Create an `AuthModule`.
3.  **Auth Service:** Implement two methods:
    *   `signup(email, password)`: Hashes the password and saves the new user.
    *   `login(email, password)`: Finds the user, compares the hashed password, and if valid, generates a JWT.
4.  **JWT Strategy:** Create a `JwtStrategy` that validates the token on incoming requests.
5.  **Auth Resolver:** Create two mutations:
    *   `signup(input)`: Returns the user and a token.
    *   `login(input)`: Returns the user and a token.
6.  **Authenticated Query:** Create a `me` query that is protected by a Guard (`@UseGuards(JwtAuthGuard)`). It should return the currently logged-in user.

> 💡 **Hint:** The `AuthModule` will need to import `JwtModule` and `PassportModule`. Read the NestJS docs on JWT authentication carefully.

---

## Phase 3: The Core Feature - Tasks & Ownership

**Objective:** Users can create and manage their own tasks.

### Your Mission:
1.  **Task Entity:** Create the `Task` entity (`id`, `title`, `description`, `status`).
2.  **The Relation (Ownership):** A `Task` belongs to **one** `User`. A `User` can have **many** `Tasks`. Implement this `One-to-Many` relationship.
3.  **Task Resolver:** Implement the following, all protected by the `JwtAuthGuard`.
    *   `createTask(input)`: Creates a new task. **Crucially, it must be linked to the currently authenticated user.**
        *   *Hint:* You'll need a way to get the current user inside the resolver. A custom decorator `@CurrentUser` is the elegant way.
    *   `myTasks()`: A query that returns only the tasks belonging to the logged-in user.
    *   `updateTask(id, input)`: A mutation to update a task. **Business Rule:** A user can only update their *own* tasks. You must check for ownership in the service layer.

> **⚠️ Pitfall:** A common mistake is to let the client send the `userId`. Never trust the client. Always get the user ID from the validated JWT token.

---

## Phase 4: Advanced Data - Tags (Many-to-Many)

**Objective:** Add complexity by allowing tasks to be categorized with multiple tags.

### Your Mission:
1.  **Tag Entity:** Create a simple `Tag` entity (`id`, `name`).
2.  **The M2M Relation:** A `Task` can have **many** `Tags`. A `Tag` can be on **many** `Tasks`.
    *   This requires a third "join table" in the database.
    *   *Hint:* Use TypeORM's `@ManyToMany` and `@JoinTable` decorators.
3.  **Resolver Logic:**
    *   `createTag(name)` mutation.
    *   Modify `createTask` and `updateTask` to accept an array of `tagIds`.
    *   The service logic must fetch the `Tag` entities and correctly associate them with the `Task`.

> 🚀 **Challenge:** When you query for tasks, also return their associated tags.

---

## Phase 5: Real-Time Interactivity - Subscriptions

**Objective:** Make the Kanban board collaborative. When one user updates a task, other users see the change instantly.

### Your Mission:
1.  **Enable Subscriptions:** Configure the `GraphQLModule` to handle subscriptions over WebSockets.
2.  **`PubSub`:** Create and provide a `PubSub` instance. This is the event bus that will broadcast changes.
3.  **Create a Subscription Resolver:**
    *   `@Subscription(() => Task, { name: 'taskUpdated' })`: This function will be called for clients listening for updates.
4.  **Publish Events:**
    *   In your `updateTask` mutation in the service, after successfully saving the change to the database, **publish** an event using `pubSub.publish()`. The payload should be the updated task.

> 🧪 **Test:** Open two GraphQL Playground windows. In one, listen to the `taskUpdated` subscription. In the other, call the `updateTask` mutation. The first window should receive the data instantly.

---

## Phase 6: Performance Tuning - The N+1 Problem (Bonus)

**Objective:** Fix a hidden performance bottleneck common in GraphQL APIs.

**The Problem:** If you query for 10 tasks, and for each task you query its owner (the user), you will make 1 (for tasks) + 10 (for each user) = **11** database calls. This is the N+1 problem.

### Your Mission:
1.  **Install `dataloader`:** `npm install dataloader`.
2.  **Create a User Loader:** Create a service/provider that uses `DataLoader`. It will take an array of user IDs and fetch them all in a single SQL query (`SELECT * FROM "user" WHERE "id" IN (...)`).
3.  **Field Resolver:** Create a `@ResolveField()` for the `user` property on the `Task` object. This resolver will use the `DataLoader` to batch and cache user requests.


---


### Good luck :-)
