# School Management System - Architecture Overview

## Tenant Model

- The system is multi-tenant
- Tenant boundary is defined by a **School**
- Each school is fully isolated using `schoolId`

> Tenant boundary = School  
> Inside each tenant exist multiple bounded contexts

---

## Tenant Management Context (Platform Level)

### Entities

- Tenant (School)
- TenantSettings

### Behaviors

- register school (create tenant)
- activate / suspend tenant

### Ownership

- Owns tenant lifecycle
- Owns global tenant registry
- Does NOT own teacher/student domain data

---

## Identity & Access Context

### Entities

- User
- Role
- Permission
- Membership (user in tenant)

### Behaviors

- authenticate user
- assign role
- grant/revoke permissions
- enforce school-level access boundaries

### Ownership

- Owns authentication and authorization
- Owns user-to-tenant association
- Does NOT own tenant provisioning lifecycle

---

## Event Management Context

### Entities

- Event (webinar, concert, competition)
- EventParticipation

### Domain Concepts

- attendance
- places / scores
- certificates / awards

### Behaviors

- create event
- register
- add link performance video/webinar
- attend
- mark attendance
- assign place
- send certificate

### Ownership

- Owns attendance lifecycle
- Owns event participation state

---

## Lesson Planning Context

### Entities

- LessonPlan
- LessonPlanAssignment

### Behaviors

- create plan
- make template
- create from template
- assign plan to student

### Ownership

- Owns lesson planning logic
- Owns assignment of plans

---

## Reporting Context (Read Model)

### Core Concept

- Report is **computed (not stored as a domain entity)**

### Inputs

- EventParticipation (Event Context)
- LessonPlanAssignment (Lesson Context)

### Behaviors

- aggregate data
- compute results
- generate reports
- combine event + lesson outcomes
- produce grades and summaries

### Ownership

- Owns aggregation logic only
- Does NOT own source data

---
