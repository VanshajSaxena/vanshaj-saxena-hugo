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

This architecture satisfies the current requirements while being scalable for the future. If the need arises, we can introduce a high-speed caching layer or decompose the system into microservices as the system expands.

## Development Process

### Backend Application

### Client Application

### Vendor Application

## Deployment

## Future
