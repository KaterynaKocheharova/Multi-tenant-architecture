# 📄 School Management System - Architecture Overview

## 🏫 Tenant Model

- The system is multi-tenant
- Tenant boundary is defined by a **School**
- Each school is fully isolated using `schoolId`

> Tenant boundary = School  
> Inside each tenant exist multiple bounded contexts

---

## 👤 Identity / School Context (Tenant Boundary Context)

### Entities

- School (Tenant)
- User
  - Teacher
  - Student
- Roles

### Responsibilities

- User management
- Role management
- Access control
- Permissions
- Tenant (School) association

### Language

- belongs to school
- access control
- permissions
- role assignment

---

## 🎯 Event Management Context

### Entities

- Event (webinar, concert, competition)
- EventParticipation

### Domain Concepts

- attendance
- places / scores
- certificates / awards

### Behaviors

- create event
- organize event
- add link performance video/webinar
- register
- attend
- mark attendance
- assign place

### Ownership

- Owns attendance lifecycle
- Owns event participation state

---

## 📚 Lesson Planning Context

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

## 📊 Reporting Context (Read Model)

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

## 🔗 Context Ownership Rules

- Event Context → attendance + participation lifecycle
- Lesson Context → assignments + lesson planning
- Reporting Context → aggregation + computed results
- Identity Context → users, roles, permissions, tenant boundary

---

## 🧠 System Summary

- Tenant = School
- Each tenant has the same bounded contexts
- Contexts are isolated but share `schoolId`
- Reporting depends on other contexts but does not modify them
- Identity Context controls access across all contexts
