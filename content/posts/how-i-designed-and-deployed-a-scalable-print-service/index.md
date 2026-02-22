---
title: "How I Designed and Deployed a Scalable, Cost Efficient Print Service Architecture"
date: 2026-02-19T15:06:43+05:30
draft: false
showToc: true
---

## Why

Printing is pretty common in educational institutions, so much so, that when I was I college we had to print hundreds of pages per subject per week. The process is not complex by any means but requires 3 different parties to communicate well.

- **WhatsApp** - This is your relay, which was never meant to solve your printing needs.
- **You (The Consumer)** - You want to print 3 documents, you want only the first 10 pages of the first document, not want only the first page of the second document and need the first two document in Black and White, but the last one in Color.
- **Printer (The Print Shop Vendor)** - This person will download your documents from WhatsApp, ask you three times which are _your_ documents (since he prints hundreds in numbers), then he will show you each document one by one to gather your requirements and print according to them.

You see where is this going? And we haven't even talked about images, they are in itself a monster (cropping, resizing and visual corrections).

I envisioned a simple service, that does just this:

{{< inline-svg src="scan-select-print.svg" >}}

I started my research and found that there are some apps trying to solve the same issue, but I didn't like any of the implementations some were hardly accessible or were too convoluted to use. I wanted something simple, that can be used by anyone (simplicity) and anywhere (accessibility).

## Design

Before I could design the system architecture, I needed to clearly define the requirements of the system.

### Requirements

I needed three systems that harmoniously integrates into a single product. These three systems were:

| System              | Role          | Responsibilities                                                               |
| ------------------- | ------------- | ------------------------------------------------------------------------------ |
| Client Application  | User Facing   | Submission of print jobs, selection of print settings, order tracking, payment |
| Vendor Application  | Vendor Facing | Printing and fulfillment of print jobs                                         |
| Backend Application | Orchestration | Business logic, system coordination, and data flow management                  |

These are some functional and non-functional requirements that I identified for the system:

#### Functional Requirements

- Users should be able to submit print jobs remotely from anywhere.
- Users should be able to upload and manage multiple documents in a single order.
- Users should be able to see the real-time availability of the printers and their proximity.
- Users should be able to track the status of their print order in real time.

#### Non-Functional Requirements

- The documents should be kept private and should be automatically deleted after printing respecting user's privacy.
- The system should withstand high traffic even during peak hours (e.g., before exams).
- The system should be cost-efficient to operate and maintain.

Below is the system architecture I designed to meet the above requirements:

{{< inline-svg src="ps-system-design.svg" >}}

This architecture satisfies the current requirements while being scalable for the future. If the need arise, we can introduce a high-speed caching layer or decompose the system into microservices as the system expands.

## Development Process

Now, that our architecture is ready, the dirty work starts, we now dive into code and implement the system in a series of steps.
I chose the following two languages for the development:

| Languege   | Scope                                                 | Rationale                                                                            |
| ---------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------ |
| TypeScript | Backend, Client App, Vendor App                       | Known for it's safety and it's powerful type system.                                 |
| C#         | Printing Layer, Windows Print Spooler API integration | We will use this to create a printing layer that talks to Windows Print Spooler API. |

### Initialization

To scaffold the application codebases I skimmed through some high-quality open-source codebases to get an idea of their structure and the practices they follow.

Unfortunately, the modern web development is a scattered mess. You need different tooling for everything and you need to know the tools and dependencies you require. Nevertheless, I settled upon these:

| Tool               | Role                                                         |
| ------------------ | ------------------------------------------------------------ |
| Yarn v4 (Berry)    | Package Manager that we will be using.                       |
| Vite (with Rollup) | The build tool/bundler to package and minify our code.       |
| Nest CLI           | For NestJS, they have their own CLI to scaffold/orchestrate. |

#### NestJS

Following instruction on the [NestJS](https://nestjs.com/) website, I had my very first controller method ready.

```ts
import { Controller, Get } from "@nestjs/common";
import { AppService } from "./app.service";

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

#### React and Electron

I will be writing a separate post on how I initialized these, these were a behemoth to get right (especially the Electron project) with the settings and dependencies I wanted.

### Dependencies and Libraries

I identified these dependencies that we will need to implement the bare architecture:

| Dependencies                    | Rationale                                                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `@aws-sdk/client-s3`            | We need this client to generate Pre-Signed URLs used by both our frontends.                                                 |
| `@aws-sdk/s3-request-presigner` | This gives us `getSignedUrl` helper that receives our client.                                                               |
| `pg`                            | This is the non-blocking PostgreSQL client that creates and maintains a pool of connections to the database.                |
| `typeorm`                       | A very advanced ORM inspired by the OG ORMs like `Hibernate`, `Doctrine` and `Entity Framework`.                            |
| `@nestjs/*`                     | Nest specific dependencies what NestJS automatically installs during scaffolding, like `@nestjs/core` and `@nestjs/common`. |

With just these dependencies installed, we can easily deploy a working prototype. But in any serious project, we need a few more dependencies to increase the development speed and elevate the DX.

| Dependencies                            | Rationale                                                                                                                                        |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `class-transformer` & `class-validator` | Instead of manually validating and transforming our DTOs we simply declare what we need.                                                         |
| `nanoid`                                | A tiny, secure, URL-friendly, unique string ID generator, we will use this to generate human-friendly identifiers, with entropy matching UUIDv4. |

### Development Flow

The development of a system like this can be broken down into few quintessential steps:

#### NestJS

##### Designing Database Schemas

- Defining the entities the system will deal with, for a printing system these will typically be (**job**, **document**, **printer**, etc.).
- This includes defining any relationships between them (the cardinality and the direction of relationships).

<details>
  <summary><strong>Click to show a code example</strong></summary>

```ts
import { Job } from "src/job/entities/job.entity";
import {
  Column,
  CreateDateColumn,
  DeleteDateColumn,
  Entity,
  OneToMany,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
} from "typeorm";
import { DocumentStatus } from "../enum/document-status.enum";

@Entity()
export class Document {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 255 })
  filename: string;

  @Column()
  key: string;

  @Column({ type: "varchar", length: 255 })
  contentType: string;

  @Column("enum", { enum: DocumentStatus, default: DocumentStatus.CREATED })
  status: DocumentStatus;

  @OneToMany(() => Job, (job) => job.document)
  jobs: Job[];

  @CreateDateColumn({ type: "timestamptz" })
  createdAt: Date;

  @UpdateDateColumn({ type: "timestamptz" })
  updatedAt: Date;

  @DeleteDateColumn({ type: "timestamptz" })
  deletedAt: Date;
}
```

</details>

##### Designing Controllers

- To let the Client and the Vendor Applications talk to our service we need to expose our entities as resources through REST endpoints.
- We create controllers for each resource that define the contract of our API and inject services that handles the exact business logic.

<details>
  <summary><strong>Click to show a code example</strong></summary>

```ts
import {
  Body,
  Controller,
  Delete,
  Get,
  Param,
  ParseIntPipe,
  Patch,
  Post,
} from "@nestjs/common";
import { DocumentService } from "./document.service";
import { CreateDocumentBatchRequestDto } from "./dto/create-document-batch-request.dto";
import { CreateDocumentBatchResponseDto } from "./dto/create-document-batch-response.dto";
import { UpdateDocumentRequestDto } from "./dto/update-document.dto";

@Controller({ path: "documents", version: "1" })
export class DocumentController {
  constructor(private readonly documentService: DocumentService) {}

  @Post()
  async create(
    @Body() createDocumentBatchDto: CreateDocumentBatchRequestDto,
  ): Promise<CreateDocumentBatchResponseDto> {
    return this.documentService.create(createDocumentBatchDto);
  }

  @Get()
  async findAll() {
    return this.documentService.findAll();
  }

  @Get(":id")
  async findOne(@Param("id", ParseIntPipe) id: number) {
    return this.documentService.findOne(id);
  }

  @Patch(":id")
  async update(
    @Param("id", ParseIntPipe) id: number,
    @Body() updateDocumentRequestDto: UpdateDocumentRequestDto,
  ) {
    return this.documentService.update(id, updateDocumentRequestDto);
  }

  @Delete(":id")
  async remove(@Param("id", ParseIntPipe) id: number) {
    return this.documentService.remove(id);
  }
}
```

</details>

##### Defining DTOs (Data Transfer Objects)

- DTOs help separate the presentation layer from the business layer.
- This is where our `class-validator` dependency comes handy. We can define the exact rules the DTOs has to validate against before it even touches our controller.

<details>
  <summary><strong>Click to show a code example</strong></summary>

```ts
import {
  ArrayMinSize,
  IsArray,
  IsNotEmpty,
  ValidateNested,
} from "class-validator";
import { CreateDocumentRequestDto } from "./create-document-request.dto";

export class CreateDocumentBatchRequestDto {
  @IsNotEmpty()
  @IsArray()
  @ArrayMinSize(1)
  @ValidateNested({ each: true })
  documents: CreateDocumentRequestDto[];
}
```

</details>

##### Building Services

- Services are responsible for defining and enforcing business rules.
- A typical service (or provider in NestJS terminology) needs to inject a TypeORM repository and perform actions on the entity object through the interface exposed by those repositories using data received from the DTOs.

<details>
  <summary><strong>Click to show a code example</strong></summary>

```ts
import { Injectable, InternalServerErrorException } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { ApplicationLogger } from "src/logger/application-logger.service";
import { Repository } from "typeorm";
import { CreateDocumentBatchRequestDto } from "./dto/create-document-batch-request.dto";
import { CreateDocumentBatchResponseDto } from "./dto/create-document-batch-response.dto";
import { CreateDocumentResponseDto } from "./dto/create-document-response.dto";
import { Document } from "./entities/document.entity";

@Injectable()
export class DocumentService {
  constructor(
    @InjectRepository(Document)
    private documentRepository: Repository<Document>,
    private logger: ApplicationLogger,
  ) {
    this.logger.setContext(DocumentService.name);
  }

  async create(createDocumentDto: CreateDocumentBatchRequestDto) {
    const draftIdMap = new Map<string, string>();
    const newDocuments = createDocumentDto.documents.map((document) => {
      const documentkey = this.getDocumentKey(document);
      const newDocument = this.documentRepository.create(document);
      newDocument.key = documentkey;
      draftIdMap.set(newDocument.key, document.draftId);
      return newDocument;
    });
    const saved = await this.documentRepository.save(newDocuments);
    const documents = saved.map((savedDocument) => {
      const draftId = draftIdMap.get(savedDocument.key);
      if (!draftId) {
        this.logger.error(
          "Document creation failed due to in-memory mapping failure",
        );
        throw new InternalServerErrorException(
          `Mapping failed for key: ${savedDocument.key}`,
        );
      }
      return new CreateDocumentResponseDto({ ...savedDocument, draftId });
    });
    return new CreateDocumentBatchResponseDto({ documents });
  }
}
```

</details>

#### React and Electron

I realized this is already getting so long, I will be creating a separate posts on how I created the React and Electron application.

### Tools

Here are some tools that I use daily during development that are worth mentioning, these incredibly hastens the process of development for me.

| Tool                                      | Description                                                    |
| ----------------------------------------- | -------------------------------------------------------------- |
| `git` + `lazygit`                         | `lazygit` is a git interface for the TUI.                      |
| `fd`                                      | Incredibly fast file search.                                   |
| `rg` (ripgrep)                            | An incredibly fast text search tool, a replacement for `grep`. |
| more tools that I am forgetting right now |                                                                |

You might have noticed, I don't use an IDE and only use the tools I need.

## Deployment

Before we deploy our application on the cloud we will first containerize our application using **Docker** to have a consistent runtime environment everytime we run our application. This reduces surprises due to the differences in the environment our application will run in.

Here, I am using the **node:22-bookwork-slim** which keeps the our container image lightweight.

```dockerfile
FROM node:22-bookworm-slim AS builder

WORKDIR /app

RUN corepack enable

COPY package.json yarn.lock ./
COPY .yarn .yarn
COPY .yarnrc.yml ./

RUN yarn install --immutable

COPY . .

RUN yarn build

FROM node:22-bookworm-slim

WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
COPY --from=builder /app/yarn.lock ./
COPY --from=builder /app/.yarn .yarn
COPY --from=builder /app/.yarnrc.yml ./

RUN corepack enable \
  && yarn install --immutable \
  && yarn cache clean --all

EXPOSE 8080

CMD ["node", "dist/main"]
```

Now, we can build:

```sh
$ docker build -t printit-cloud .
```

And, run:

```sh
$ docker run --env-file .env.development printit-cloud
```

With this, the docker image can easily be used to run the application on any cloud provider.

## Conclusion

This is yet to complete and by far perfect. I aim to improve this gradually as the requirements of the system becomes more clear and the system expands. I will also be discussing the architecture of the two other applications we left in upcoming future posts.
