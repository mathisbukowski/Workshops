# Workshop: Professional E-Commerce API with NestJS
**Topic:** Relational Data, Swagger Documentation & Business Logic
**Duration:** 3 Hours
**Goal:** Build a production-ready API managing **Products** and **Categories** with a relationship system.

---

## Objectives
By the end of this workshop, you will understand:
1.  **NestJS Architecture:** Modules, Controllers, Services.
2.  **TypeORM Relations:** How to link tables (One-to-Many) correctly.
3.  **DTO & Validation:** Professional input checking.
4.  **OpenAPI (Swagger):** Generating automatic documentation for frontend developers.

---

## Part 1: Project Setup

### 1.1 Initialization
Create a new project using the NestJS CLI.

```bash
npm i -g @nestjs/cli
nest new shop-api
# Select 'npm'
cd shop-api
```

### 1.2 Install Professional Dependencies
We need TypeORM (Database), SQLite (Driver), Validation tools, and **Swagger** (Documentation).

```bash
npm install --save @nestjs/typeorm typeorm sqlite3 class-validator class-transformer @nestjs/swagger swagger-ui-express
```

### 1.3 Database Connection
Configure the `app.module.ts` to connect to a SQLite file.

```typescript
// app.module.ts
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'sqlite',
      database: 'shop.db',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: true, // ⚠️ Auto-creates tables (Dev only)
    }),
  ],
})
export class AppModule {}
```

---

## Part 2: The "Category" Resource

We start with the parent entity. A Category has a name and many products.

### 2.1 Generate Resources
Use the CLI to generate everything for Categories.
```bash
nest g resource categories
# Select "REST API" -> "Yes" for CRUD entry points
```

### 2.2 Define the Entity
Open `src/categories/entities/category.entity.ts`.

```typescript
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Product } from '../../products/entities/product.entity'; // Will be created later

@Entity()
export class Category {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  // Placeholder for the relationship (we will uncomment this later)
  // @OneToMany(() => Product, (product) => product.category)
  // products: Product[];
}
```

### 2.3 Implement Service & Controller
**Goal:** Implement `create` and `findAll` using `InjectRepository`.
*   Inject `Repository<Category>` in the service.
*   Use `.save()` and `.find()`.

> **Check:** Test with Postman (POST /categories) to verify you can create a "Electronics" category.

---

## Part 3: Products & Relations (The Core)

This is where it gets interesting. A Product belongs to a Category.

### 3.1 Generate Products
```bash
nest g resource products
# Select "REST API" -> "Yes"
```

### 3.2 The Product Entity (Child)
Open `src/products/entities/product.entity.ts`.

```typescript
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from 'typeorm';
import { Category } from '../../categories/entities/category.entity';

@Entity()
export class Product {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column('decimal')
  price: number;

  @Column({ default: 0 })
  stock: number;

  // The Magic: Linking to Category
  @ManyToOne(() => Category, (category) => category.products, { onDelete: 'CASCADE' })
  category: Category;
}
```

### 3.3 Update Category Entity (Parent)
Go back to `category.entity.ts` and uncomment the `@OneToMany` part.
You need to import the `Product` entity.

### 3.4 Register Entities
Make sure both `TypeOrmModule.forFeature([Category])` and `TypeOrmModule.forFeature([Product])` are in their respective Modules.

---

## Part 4: DTOs with Relationship Logic

When creating a product, we want to send the `categoryId`.

### 4.1 Create Product DTO
Open `src/products/dto/create-product.dto.ts`.

```typescript
import { IsString, IsNumber, IsPositive, IsInt } from 'class-validator';

export class CreateProductDto {
  @IsString()
  name: string;

  @IsNumber()
  @IsPositive()
  price: number;

  @IsInt()
  stock: number;

  @IsInt()
  categoryId: number; // We transfer the ID, not the full object
}
```

### 4.2 The "Smart" Creation Logic
In `products.service.ts`, we cannot just save the DTO. We must link the relationship.

```typescript
// products.service.ts
constructor(
  @InjectRepository(Product) private productRepo: Repository<Product>,
  // We explicitly inject Category Repository to check if it exists!
  @InjectRepository(Category) private categoryRepo: Repository<Category>,
) {}

async create(createProductDto: CreateProductDto) {
  // 1. Check if category exists
  const category = await this.categoryRepo.findOneBy({ id: createProductDto.categoryId });

  if (!category) {
    throw new NotFoundException('Category not found');
  }

  // 2. Create product and assign the relationship object
  const newProduct = this.productRepo.create({
    ...createProductDto,
    category: category, // TypeORM needs the object, not just the ID
  });

  return this.productRepo.save(newProduct);
}
```
*Note: You will need to export `CategoryService` or inject the repository properly.*

---

##  Part 5: Swagger Documentation (Pro Touch)

We want a professional documentation page for our API.

### 5.1 Main Setup
In `main.ts`:

```typescript
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger Config
  const config = new DocumentBuilder()
    .setTitle('Shop API')
    .setDescription('The best e-commerce API')
    .setVersion('1.0')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document); // Docs available at /api

  await app.listen(3000);
}
bootstrap();
```

### 5.2 Decorating DTOs
Go back to `create-product.dto.ts` and add `@ApiProperty()` to fields.

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateProductDto {
  @ApiProperty({ description: 'The name of the product', example: 'iPhone 15' })
  @IsString()
  name: string;

  // ... do the same for others
}
```

**Result:** Restart the server and go to `http://localhost:3000/api`. You now have an interactive documentation site automatically generated!

---

## ⚛️ Part 6: The "Senior" Challenge - Hybrid GraphQL API

**Context:** The mobile team complains that the REST API sends too much data. They only want the `name` and `price`, not the `stock` or `id`.
**Goal:** Expose the existing Products via GraphQL to allow flexible fetching.

### 6.1 Install GraphQL Dependencies
NestJS makes GraphQL setup easy, but there are many packages. We will use the **Code First** approach (generating the schema from TypeScript classes).

```bash
npm install @nestjs/graphql @nestjs/apollo graphql apollo-server-express
```

### 6.2 Configure the Module
Add `GraphQLModule` to `app.module.ts`.

```typescript
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { join } from 'path';

@Module({
  imports: [
    // ... TypeOrmModule is here
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'), // Automatically generates the schema file
      sortSchema: true,
      playground: true, // Enables the interactive query tool
    }),
    // ... Modules
  ],
})
export class AppModule {}
```

### 6.3 Decorate the Entity (Code First)
Instead of creating a separate type, we will transform our existing `Product` entity into a GraphQL Object Type.
Open `src/products/entities/product.entity.ts`.

```typescript
import { ObjectType, Field, Int, Float } from '@nestjs/graphql'; // <--- Import this
// ... existing imports

@ObjectType() // <--- 1. Mark class as GraphQL Type
@Entity()
export class Product {
  @Field(() => Int) // <--- 2. Expose this field to GraphQL
  @PrimaryGeneratedColumn()
  id: number;

  @Field() // Default is String
  @Column()
  name: string;

  @Field(() => Float)
  @Column('decimal')
  price: number;

  // We intentionally DO NOT add @Field() to 'stock'.
  // It will remain secret/internal to the REST API!
  @Column({ default: 0 })
  stock: number;

  @Field(() => Category) // We can even expose relations!
  @ManyToOne(() => Category, (category) => category.products)
  category: Category;
}
```
*Note: Do the same for `Category` entity (add `@ObjectType` and `@Field` on name/id).*

### 6.4 Create the Resolver
In GraphQL, "Controllers" are called "Resolvers".
Create `src/products/products.resolver.ts`.

```typescript
import { Resolver, Query, Args, Int } from '@nestjs/graphql';
import { ProductsService } from './products.service';
import { Product } from './entities/product.entity';

@Resolver(() => Product)
export class ProductsResolver {
  constructor(private readonly productsService: ProductsService) {}

  @Query(() => [Product], { name: 'getAllProducts' })
  findAll() {
    return this.productsService.findAll(); // Reuse the logic from the Service!
  }

  @Query(() => Product, { name: 'getProduct' })
  findOne(@Args('id', { type: () => Int }) id: number) {
    return this.productsService.findOne(id);
  }
}
```

### 6.5 Register the Resolver
Don't forget to add `ProductsResolver` to the `providers` array in `products.module.ts`.

```typescript
@Module({
  // ...
  providers: [ProductsService, ProductsResolver],
})
export class ProductsModule {}
```

### 6.6 Test the Hybrid API
You now have a server running REST **AND** GraphQL simultaneously!

1.  Open your browser at `http://localhost:3000/graphql`.
2.  Write a query to ask **only** for names and prices (no stock, no ID):

```graphql
query {
  getAllProducts {
    name
    price
    category {
        name
    }
  }
}
```

**Why is this difficult?**
*   You need to understand two paradigms (REST vs Graph).
*   You are reusing the *Service* layer for both inputs (Controller and Resolver). This demonstrates the power of clean architecture: **Logic is separated from the transport layer.**


---

##  Part 7: Quality Assurance & Unit Testing (Bonus)

**Context:** In a professional environment, you cannot push code without tests. NestJS generates `.spec.ts` files automatically. Let's use them.
**Goal:** Write a unit test for the `ProductsService` without touching the real database (Mocking).

### 7.1 Understanding the Spec file
Open `src/products/products.service.spec.ts`.
You will see a `beforeEach` block. This creates a "fake" module just for testing.

**Problem:** If you run `npm test` now, it will fail because the Service expects a Database Repository, but we didn't provide one in the test module.

### 7.2 Mocking the Repository
We need to tell Jest: "When the service asks for the Repository, give it this fake object instead."

Update `src/products/products.service.spec.ts`:

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ProductsService } from './products.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Product } from './entities/product.entity';
import { Category } from '../categories/entities/category.entity';

describe('ProductsService', () => {
  let service: ProductsService;

  // 1. Create a "Mock" (a fake version of the repository)
  const mockProductRepo = {
    find: jest.fn().mockImplementation(() => Promise.resolve([{ id: 1, name: 'Test Product' }])),
    create: jest.fn(),
    save: jest.fn(),
  };

  const mockCategoryRepo = {
    findOneBy: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ProductsService,
        // 2. Provide the Mock instead of the real Repository
        {
          provide: getRepositoryToken(Product),
          useValue: mockProductRepo,
        },
        {
          provide: getRepositoryToken(Category),
          useValue: mockCategoryRepo,
        },
      ],
    }).compile();

    service = module.get<ProductsService>(ProductsService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  // 3. Write the actual test
  it('findAll should return an array of products', async () => {
    const result = await service.findAll();

    // Assertions
    expect(result).toBeInstanceOf(Array);
    expect(result[0].name).toEqual('Test Product');
    expect(mockProductRepo.find).toHaveBeenCalled(); // Check if the function was called
  });
});
```

### 7.3 Run the Tests
Execute the command:
```bash
npm run test
```
You should see green text confirming that your logic is valid!

---

## Troubleshooting & Tips

*   **Circular Dependency:** If `Category` imports `Product` and `Product` imports `Category`, it might crash. use `forwardRef()` if needed, but usually TypeORM handles it if you use `type => Product`.
*   **Validation not working?** Don't forget `app.useGlobalPipes(new ValidationPipe())` in `main.ts`.
*   **"Repository not found":** Ensure `TypeOrmModule.forFeature([Entity])` is in the module imports.
