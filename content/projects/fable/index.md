---
title: 'Fable'
#date: 2024-02-10T03:36:02+05:30
draft: false
showtoc: false
weight: 99
---
---

Fable is a Software as a Service (SaaS) Library Management System (LMS)
Software.

I worked on Fable as a part of an Internship at Infosys Limited,
Mysuru, which I received through a challenging selection process, I have a post
about it, [here]({{< relref "posts/got-selected" >}}).

Here are the videos of the actual developed applications.

---

{{< video src="FableForMembersandUsers.mp4" width="300" height="450" class="align-center" >}}

{{< video src="FableForAdminsandLibrarians.mp4" width="600" height="450" class="align-center" >}}

---
## Development Overview

The Fable project is subdivided into two application projects, one is for the
users and the other is only for library management (mainly librarians) and
administrators. I worked as the **backend architect** and **developer**.

### User LMS

[Source code](https://github.com/VanshajSaxena/LMS_User)

#### Layout
``` shell
LMS_User
├── Assets.xcassets # Assets folder
├── CONTRIBUTING.md # Contributing Guide
├── GoogleService-Info.plist  # FireBase Integration
├── LMSUser
│   ├── Info.plist
│   ├── LMSUser.swift
│   ├── Models # Models and Entities
│   ├── Services # Services that interact with APIs
│   └── Views # A lot of SwiftUI Views, basically the UI
├── LMSUser.xcodeproj # XCode specific folder, But I like to tinker in
│   ├── project.pbxproj # The project index file used by the source-kit LSP
│   ├── project.xcworkspace # Workspace settings, specific to user environment and settings
│   │   ├── contents.xcworkspacedata
│   └── xcshareddata
├── 'Preview Content'
│   └── 'Preview Assets.xcassets'
│       └── Contents.json
└── README.md # Readme file
```
The application follows a simple MVVM architecture to structure the source
code.

### Admin Librarian LMS

[Source code](https://github.com/VanshajSaxena/LMS_Admin_Librarian)

``` shell
LMS_Admin_Librarian
├── CONTRIBUTING.md
├── GoogleService-Info.plist
├── LMS-Admin-Librarian-Info.plist
├── LMSAdminLibrarian
│   ├── Assets.xcassets
│   ├── Models # Entities and models
│   ├── 'Preview Content'
│   │   └── 'Preview Assets.xcassets'
│   │       └── Contents.json
│   ├── Services # Services that interact with APIs
│   ├── Utilities # Utility classes and functions
│   ├── ViewModels # View Models for SwiftUI views
│   └── Views # SwiftUI views files
├── LMSAdminLibrarian.xcodeproj # XCode specific folder
│   ├── project.pbxproj # Project index required by LSP
│   └── project.xcworkspace
│       ├── contents.xcworkspacedata
│       └── xcshareddata
└── README.md # Readme file
```

I mainly worked on the majorly on the Administrator's and Librarian's
application. At the time of starting I knew very little about applications
programming, I learned through the process of developing and sharing.

---

I learned a lot from the project as this was my first project in which I worked
with a team of 10 members in an agile-based development process. I also had the
responsibility of the Scrum Master for the team.

