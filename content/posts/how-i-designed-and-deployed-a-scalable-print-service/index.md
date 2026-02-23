---
title: "How I Designed and Deployed a Scalable, Cost Efficient Print Service Architecture"
date: 2026-02-19T15:06:43+05:30
draft: false
showToc: true
---

## Why

Printing is pretty common in educational institutions, so much so, that when I was in college we had to print hundreds of pages per subject per week. The process is not complex by any means but requires 3 different parties to communicate well.

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

For the first milestone of the system, I have decided to only support PDF printing. When this gets stable we can add support for other formats like images and word documents. This keeps the system simple and practical without introducing unnecessary complexity in the early stages of development.

Below is the system architecture I designed to meet the above requirements:

{{< inline-svg src="ps-system-design.svg" >}}

<br>

This system alone should be able to handle most of what we need. You might notice, there is not depiction of file processing queues or worker nodes, this is intentional.

To handle the cost efficiency requirement, I wanted to offload any CPU intensive tasks to the client applications, this allows to save on server costs and also reduces the latency of the system. This comes with trade-offs, we need to ensure that the client applications are feature rich, have offline functionality (so the user can make edits even without a network, and sync changes when the network reconnects) and are capable enough to handle the processing tasks.

We usually want this kind of behavior in a system that handles printing, since letting the server manipulate the documents after they have been uploaded and sent for printing can produce mismatched results deviating from the user's expectations. By offloading the processing to the client, we can ensure that the user has full control over how their documents are processed and printed, and they can see the results before they are sent for printing.

Our architecture remains lean and simple, and our system only needs to orchestrate the flow of data between the clients and the printers, without needing to handle any of the processing tasks. Allowing us to keep our server costs low and while also reducing the latency of the system.

## Development Process

Now, that our architecture is ready, the dirty work starts, we now dive into code and implement the system in a series of steps.
I chose the following two languages for the development:

| Language   | Scope                                                 | Rationale                                                                            |
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

Well, the implementation details deserves a dedicated post, the development cycle of a system like this can be broken down into few quintessential steps:

#### 1. Database Architecture

This phase is about designing the database schemas and entity relationships. Deciding how information is stored, organized, and retrieved so that the system remains fast and reliable as it scales.

#### 2. API Contracts

This is about defining interfaces and API schemas that would establish a clear contract between the clients and the server. I focus on designing APIs that scale well and don't need breaking changes as the system evolves.

#### 3. Application Layer

This is where we build the core logic that solves the real problem. We usually define the controllers and the services that implement our API contract and business rules during this phase of the cycle. This keeps getting improved as the system evolves and requirements become more clear.

#### 4. Decoupled Development

Now, as we have the API contract, the frontends and the backend can now evolve independently, allowing for faster iterations and easier development cycles avoiding breaking changes (Although, that is not important until the project hits it's first milestone).

### Tools

These are some tools that I use daily and are worth mentioning, these are a huge time-saver:

| Tool           | Description                                                       |
| -------------- | ----------------------------------------------------------------- |
| `lazygit`      | `lazygit` is a git interface for the TUI.                         |
| `fd`           | Incredibly fast file search. Replacement for `find`.              |
| `rg` (ripgrep) | An incredibly fast text search tool, a replacement for `grep`.    |
| `zoxide`       | Rust based `cd` replacement. Helps navigating directories faster. |
| `fzf`          | Fuzzy finder.                                                     |

There are more which I am forgetting, but these are the most useful.

## Deployment

Before we deploy our application on the cloud we will first containerize our application using **Docker** to have a consistent runtime environment every time we run our application. This reduces surprises due to the differences in the environment our application will run in.

> **Containerization** essentially packages the application code, its dependencies, and the runtime into a single image that can be used to create containers that guarantees consistent behavior and execution across a variety of environments (Linux, MacOS, CI/CD, production).

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

Now, we can build our image:

```sh
$ docker build -t printit-cloud .
```

And, run it as a container (docker will automatically create a container from the specified image):

```sh
$ docker run --env-file .env.development printit-cloud
```

We can publish this image to a container registry like **Docker Hub** or **AWS ECR** and then deploy it on a cloud platform like **AWS ECS** or **AWS EKS**.

## Conclusion

This is a start. My goal is to iteratively refine and evolve the system as it scales and as the requirements becomes more clear. I’ll continue sharing updates on the development and deployment process as the project progresses.

I will be discussing the architecture of the two other applications we left, in the upcoming posts.
