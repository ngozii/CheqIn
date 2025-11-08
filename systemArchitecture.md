# CheqIn System Architecture Document (SAD)

## Document Control

| Field | Detail |
|-------|--------|
| **Project Name** | CheqIn – GPS-Based Automated Attendance System |
| **Document Version** | 1.0 |
| **Last Updated** | November 8, 2025 |
| **Author** | Mercy Ohabuike |
| **Reviewed By** | — |
| **Approved By** | — |
| **Document Status** | Draft |
| **Confidentiality Level** | Internal |

---

## 1. Introduction

### 1.1 Purpose
This System Architecture Document (SAD) provides a detailed overview of the technical architecture, components, integrations, and data flow of the CheqIn Platform. It serves as the primary reference for developers, architects, and DevOps engineers to understand how system components interact, ensuring scalability, modularity, and maintainability.

### 1.2 Scope
The document describes:
- Core architectural design and guiding principles
- Technology stack and system layers
- Integration with third-party APIs and services
- Deployment, scalability, and security architecture
- Monitoring, disaster recovery, and future roadmap

### 1.3 Intended Audience

| Role | Purpose |
|------|---------|
| **Software Architects** | Review and validate design patterns |
| **Developers** | Understand integration points and modules |
| **DevOps Engineers** | Implement CI/CD and deployment pipelines |
| **QA Teams** | Ensure environment consistency |
| **Security Teams** | Verify compliance and system hardening measures |
| **Business Stakeholders** | Reference for scalability and technology strategy |

---

## 2. System Overview

The CheqIn Platform is a cloud-native, GPS-based attendance system enabling businesses to automate employee attendance tracking through geofencing technology. It combines real-time monitoring, analytics, and automated check-in/check-out under a secure and scalable architecture.

**Key Capabilities:**
- Automated employee check-in/check-out via GPS geofencing
- Real-time office presence tracking and dashboards
- Third-party integrations (Firebase, AWS S3, SendGrid)
- Anti-spoofing detection and security monitoring
- Comprehensive attendance reporting and analytics

---

## 3. Architectural Goals and Principles

| Principle | Description |
|-----------|-------------|
| **Scalability** | System scales horizontally to support 500+ employees per organization |
| **Availability** | Designed for 99.5% uptime through redundancy and failover |
| **Security** | Multi-layer anti-spoofing detection and encrypted data storage |
| **Maintainability** | Modular services enable independent updates |
| **Extensibility** | Future-ready for Bluetooth hardware, biometrics, and AI analytics |

---

## 4. High-Level Architecture

### 4.1 Architecture Pattern
**Type:** Three-tier microservices-oriented architecture

**Layers:**
- **Presentation Layer** – React Native mobile app and React web dashboard
- **Application Layer** – Node.js + Express REST APIs
- **Data Layer** – PostgreSQL database with GeoJSON support
- **Integration Layer** – APIs for Firebase FCM, AWS S3, SendGrid
- **Infrastructure Layer** – Railway and Vercel cloud services

### 4.2 System Context Diagram

<p align="center">
<img src="CheqIn System Architecturee.drawio.png"/>
</p>

**External entities:** Employees, Admins, HR Managers

**System:** CheqIn Platform (Mobile App + Admin Dashboard + Backend API)

**External systems:** Firebase Cloud Messaging, AWS S3, SendGrid, geo-toolkits library

---

## 5. Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Mobile App** | React Native, TypeScript | Cross-platform native mobile development |
| | React Context API | State management |
| | React Native Geolocation | GPS tracking services |
| | Firebase Cloud Messaging | Push notifications |
| **Admin Dashboard** | React 18, TypeScript, Tailwind CSS | Responsive web interface |
| | Recharts | Data visualization |
| | Leaflet.js | Interactive geofence map editor |
| **Backend** | Node.js, Express.js, TypeScript | REST API server |
| | geo-toolkits | Geofencing calculations |
| | JWT, bcrypt | Authentication and security |
| **Database** | PostgreSQL 15 | Relational database |
| | Prisma ORM | Type-safe database access |
| **Cloud Infrastructure** | Railway | Backend and database hosting |
| | Vercel | Frontend hosting with CDN |
| | AWS S3 | Report file storage |
| **CI/CD** | GitHub Actions | Continuous integration and deployment |
| **Logging & Monitoring** | Sentry | Error tracking and monitoring |
| **Messaging & Alerts** | Firebase FCM, SendGrid | Push notifications and email alerts |

---

## 6. System Components

### 6.1 Frontend (Presentation Layer)

**Mobile App:**
- Framework: React Native + TypeScript
- State Management: React Context API
- API Communication: Axios for REST

**Key Features:**
- Background GPS tracking every 30 seconds
- Automatic check-in/check-out detection
- Push notification display
- Personal attendance history view
- Offline request queuing

**Responsibilities:**
- Handle all employee interactions
- Monitor GPS location in background
- Display attendance status and history
- Securely communicate with backend APIs

**Admin Dashboard:**
- Framework: React + TypeScript
- Styling: Tailwind CSS
- Charts: Recharts

**Key Features:**
- Real-time office presence indicators
- Employee management (CRUD operations)
- Interactive geofence map editor
- Analytics dashboard with trends
- CSV/PDF report generation

### 6.2 Backend (Application Layer)

**Framework:** Node.js with Express and TypeScript

**Architecture:** Modular and event-driven

**Core Service Modules:**

| Module | Description |
|--------|-------------|
| **Authentication Service** | Manages JWT tokens, sessions, and roles |
| **Attendance Service** | Handles check-in/out logic and validation |
| **Geofence Service** | GPS boundary detection using geo-toolkits |
| **Anti-Spoofing Service** | Validates location data for fraud detection |
| **User Service** | Manages employees and permissions |
| **Office Service** | Manages office locations and geofences |
| **Analytics Service** | Generates attendance metrics and reports |
| **Notification Service** | Handles push notifications and email alerts |

### 6.3 Database Layer

**Database:** PostgreSQL 15 (Managed by Railway)

**Core Tables:**
- `users`, `offices`, `departments`, `attendance`, `devices`, `security_events`

**Indexes:** Optimized for `user_id`, `date`, `timestamp`

**Backup Policy:**
- Automated daily backups
- 30-day retention period
- Point-in-time recovery capability

### 6.4 Authentication & Authorization

Implements **JWT authentication** for stateless security.

Supports **Role-Based Access Control (RBAC):**
- **Admin:** Full system access
- **Manager:** Project and team management
- **Employee:** View personal attendance only

**Security Controls:**
- bcrypt password hashing (10 rounds)
- HTTPS/TLS 1.3 enforcement
- CORS for restricted origins
- Rate limiting (100 requests per 15 minutes)

### 6.5 Integration Layer

| Integration | Function |
|-------------|----------|
| **Firebase FCM** | Push notification delivery to mobile devices |
| **AWS S3** | Storage for exported CSV/PDF reports |
| **SendGrid** | Transactional email (invitations, alerts) |
| **geo-toolkits** | Point-in-polygon geofencing calculations |

---

## 7. Data Flow and Process

**Workflow Summary:**

1. Employee arrives at office → GPS detects location
2. Mobile app sends coordinates → Backend validates via anti-spoofing
3. Backend checks geofence using geo-toolkits → Records attendance in PostgreSQL
4. Push notification sent → Employee receives confirmation
5. Admin views dashboard → Real-time presence updated
6. Analytics aggregated → Reports generated on demand

---

## 8. Deployment Architecture

### 8.1 Environments

| Environment | Purpose | Infrastructure |
|-------------|---------|----------------|
| **Development** | Local testing, containerized setup | Docker |
| **Staging** | Internal QA and validation | Railway + Vercel staging |
| **Production** | Public, high-availability deployment | Railway Pro + Vercel + AWS S3 |

### 8.2 CI/CD Pipeline

- Triggered on GitHub commit events
- Executes Jest unit and integration tests
- Staging deployment auto-triggered on success
- Production deployment upon manual approval
- Zero-downtime deployments via Railway

---

## 9. Scalability and Performance

| Focus Area | Strategy |
|-----------|----------|
| **Load Balancing** | Railway load balancer distributes requests |
| **Caching** | Future: Redis for geofence and session data |
| **Async Processing** | Background jobs for report generation |
| **Auto-Scaling** | Railway autoscaling policies |
| **Database Optimization** | Indexing, query optimization, connection pooling |

**Current Capacity:**
- 500 employees per organization
- 100 requests/second
- 10,000 daily check-ins

**Scaling Path:**
- Phase 1 (2,000 employees): Add Redis caching
- Phase 2 (10,000+ employees): Horizontal backend scaling with load balancer

---

## 10. Security Architecture

- **HTTPS enforced** across all services
- **AES-256 encryption** at rest, TLS 1.3 in transit
- **Multi-layer anti-spoofing:** IP validation, speed analysis, GPS accuracy checks
- **Role-based access control** (RBAC)
- **Rate limiting** to prevent abuse
- **Security audits** conducted quarterly

**Compliance:**
- GDPR-compliant data retention
- Location tracking only within office boundaries
- Right to access and deletion for users

---

## 11. Monitoring and Observability

| Tool | Purpose |
|------|---------|
| **Sentry** | Application error tracking |
| **Railway Metrics** | Infrastructure monitoring (CPU, memory, API response time) |
| **Custom Alerts** | Automated escalation for downtime or failed check-ins |

**Key Metrics Tracked:**
- API uptime (target: 99.5%)
- Response time (target: <200ms p95)
- Check-in success rate (target: >95%)
- GPS spoofing attempts
- Battery drain (<3% per hour)

---

## 12. Disaster Recovery and Business Continuity

| Parameter | Strategy |
|-----------|----------|
| **Backups** | Daily database snapshots, 30-day retention |
| **Failover** | Railway automatic failover mechanisms |
| **RTO (Recovery Time Objective)** | 2 hours |
| **RPO (Recovery Point Objective)** | 24 hours |
| **Testing** | Quarterly disaster recovery drills |

---

## 13. Future Enhancements

**Phase 1 (MVP):** ✅ GPS geofencing, mobile apps, admin dashboard, real-time presence, CSV export

**Phase 2:** Multi-location support, department policies, shift scheduling, email digests

**Phase 3:** Bluetooth proximity readers (hybrid GPS+Bluetooth), biometric verification, advanced analytics

**Phase 4:** White-label solution, visitor management, API marketplace, machine learning fraud detection

---

## 14. Compliance and Quality Standards

- **IEEE 1471 / ISO/IEC/IEEE 42010** – System and Software Architecture Description
- **ISO/IEC 27001** – Information Security Management
- **OWASP Top 10** – Web Application Security Principles
- **GDPR** – Data privacy and user consent

---

## 15. Appendices

### 15.1 Acronyms

| Term | Definition |
|------|------------|
| **API** | Application Programming Interface |
| **CI/CD** | Continuous Integration / Continuous Deployment |
| **JWT** | JSON Web Token |
| **RBAC** | Role-Based Access Control |
| **GPS** | Global Positioning System |
| **FCM** | Firebase Cloud Messaging |

### 15.2 Related Documents

- Product Requirements Document
- API Documentation
- User Guide
- Deployment Guide
- Privacy Policy

---

**End of Document**