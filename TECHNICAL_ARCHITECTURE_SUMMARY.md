# SmartService - Technical Architecture Summary

## Document Source
**File**: `tesis_texto_utf8.txt`  
**Document Title**: Plataforma Inteligente para la Gestión de Reclamos, Incidencias y Servicios (Intelligent Platform for Claims, Incidents and Services Management)  
**Author**: Zavatti, Santiago Nahuel  
**Date**: 04/05/2026

---

## 1. SYSTEM ARCHITECTURE OVERVIEW

### Architecture Paradigm
**Microservices-based Architecture** with centralized API Gateway

**Key Design Principle**: 
> "El sistema será diseñado bajo una arquitectura basada en microservicios, con un API Gateway central como punto principal de entrada y distintos servicios independientes encargados de resolver responsabilidades específicas."

**Rationale**: 
- Build a scalable, modular, maintainable and adaptable solution for different municipalities
- Separate responsibilities between modules
- Allow each service to evolve independently
- Reduce coupling between components
- Facilitate future system expansion

---

## 2. TECHNOLOGY STACK

### Backend Framework
**NestJS** - Primary framework for backend and microservices development
- Enables modular architecture
- Supports TypeScript
- Facilitates scalability

### API Gateway
**Centralized API Gateway** (built with NestJS)
- **Primary Entry Point**: Single access point for all clients
- **Responsibilities**:
  - Authentication (JWT-based)
  - Authorization (role-based access control)
  - Request routing to appropriate microservices
  - Initial validation
  - Security enforcement
  - Prevents direct exposure of internal microservices
  - Acts as intermediary for new services without affecting clients

### Frontend Technologies
**Client-side Stack**:
- **Framework**: Next.js
- **UI Library**: React
- **Styling**: Tailwind CSS
- **State Management**: React Query
- **Authentication**: JWT tokens

---

## 3. MICROSERVICES ARCHITECTURE

### Core Microservices

| Microservice | Primary Responsibility | Key Functions |
|---|---|---|
| **API Gateway** | Central communication point | Authentication, Authorization, Routing, Validation, Security, Centralization |
| **Claims/Reclamos** | Claim lifecycle management | Creation, Classification, Assignment, Status tracking, History, Evidence management, Queries |
| **AI/Inteligencia Artificial** | Intelligent processing | Message interpretation, Claim classification, Missing data detection, Priority suggestion, Duplicate detection |
| **Routes/Rutas** | Route optimization and geolocation | Geolocation, Territorial grouping, Route optimization via Google Maps |
| **Notifications** | Communication management | Outbound communications to citizens and internal users via WhatsApp, Email, and other channels |
| **Files/Archivos** | File management | Upload, Storage, Recovery of images, documents, evidence |
| **Audit/Auditoría** | System activity logging | Records relevant actions, state changes, accesses, critical operations |
| **Authentication & Users** | User and access management | Users, Roles, Permissions, Credentials, Access tokens |
| **Municipal Configuration** | Client-specific settings | Municipalities, Areas, Categories, Neighborhoods, Parameters, Custom rules |

---

## 4. COMMUNICATION PROTOCOLS

### Inter-Service Communication
**Technology**: NATS (lightweight, efficient messaging system)

**Communication Style**: Asynchronous event-driven messaging

**Benefits**:
- Decouples services
- Improves scalability
- Non-blocking operations
- Event-driven architecture

### Example Event Flow
When a claim is created:
```
Event: "Reclamo creado" (Claim Created)
  ↓
AI Microservice → Analyze category, priority, duplicates
  ↓
Notifications MS → Send confirmation to citizen
  ↓
Audit MS → Register change in history
  ↓
Routes MS → Include in possible routes
```

### Work Queues
**Technology**: Redis + BullMQ

**Use Cases**:
- Notification sending (with retry capability)
- Image processing (background execution)
- AI analysis (prevents blocking claim creation)
- Report generation (time-consuming processing)
- External service synchronization (handles temporary failures)
- WhatsApp message processing (maintains order, prevents event loss)

---

## 5. DATABASE ARCHITECTURE

### Relational Database - MySQL
**Use Cases**: Structured, transactional information

**Data Stored**:
- Municipalities (client data)
- Users (internal system users)
- Roles and Permissions (access control)
- Municipal Areas/Departments
- Claim Categories
- Claims (main data)
- Claim States (lifecycle)
- History (relevant actions)
- Routes (generated routes and associated claims)

**Rationale**: Maintains clear relationships, data integrity, structured queries

### Document Database - MongoDB
**Use Cases**: Unstructured, flexible information

**Data Stored**:
- WhatsApp conversations (message history)
- AI Results (responses, suggestions, analysis, metadata)
- Enriched Logs (flexible audit information)
- Semi-structured data (variable content, no rigid schema)

**Rationale**: Flexibility for conversational data and variable content

### File Storage
**Technology**: Wasabi Storage (S3-compatible)

**Advantages**:
- External to databases
- Prevents database overload
- Easier administration
- Scalable

**File Types**:
- Citizen-provided photographs (problem evidence)
- Resolution evidence (post-intervention photos)
- Attached documents (reports, certificates, attachments)
- System-generated files (reports, documents)

---

## 6. EXTERNAL INTEGRATIONS & APIs

### WhatsApp Business API
**Purpose**: Primary citizen claim channel
**Capabilities**:
- Receive citizen messages
- Send automated or assisted responses
- Request missing data
- Confirm claim creation
- Notify status changes
- Provide tracking codes

**Strategic Importance**: 
> "Reduces adoption barrier - no need for specific app download or complex portal access. Interaction through daily-use tool facilitates citizen participation."

### Google Maps Platform
**Use Cases**:
- Geocoding (convert addresses to coordinates)
- Map visualization (display claims on territory)
- Route calculation (compute routes between points)
- Route optimization (order points for efficient paths)

### Artificial Intelligence Services
**Functions**:
- Message interpretation and analysis
- Claim classification
- Duplicate detection
- Priority suggestion
- Preliminary response generation

**Technology Option**: Python for specialized analysis, processing, or complementary models integrated with main architecture

---

## 7. SECURITY CONSIDERATIONS

### Authentication & Authorization
**Authentication Method**: JWT (JSON Web Tokens)
- Validates user identity
- Validates sessions on protected operations

**Authorization Approach**: Role-based Access Control (RBAC)

**User Roles with Access Levels**:
- **Operator**: Load, review, and route claims
- **Area Manager**: Manage claims assigned to their department
- **Official**: View reports and indicators
- **Municipal Administrator**: Manage users, areas, categories
- **Platform Administrator**: Global configuration and support

### Data Protection
**Protected Data**:
- Personal names
- Telephone numbers
- Addresses
- Locations
- Images
- Contact information
- Municipal internal data

**Access Control**: Information accessible only to authorized users

### Audit Trail
**Tracked Activities**:
- Claim creation
- State changes
- Priority modifications
- Assignments
- Closures/Rejections
- Evidence uploads
- Route generation
- Configuration changes

### File Security (Wasabi Storage)
- Secure access mechanisms
- Prevents unnecessary public exposure
- Controlled user access to evidence and documents

### Backup and Recovery Policies

**Frequency Strategy**:

| Frequency | Purpose | Scope |
|---|---|---|
| **Daily Backup** | Short-term recovery from recent errors | Full data snapshot |
| **Weekly Backup** | Medium-term recovery from larger incidents | Consolidated copy |
| **Monthly Backup** | Long-term conservation | Historical copy |

**Three-Media Criterion** (Risk Mitigation):
1. **Primary Platform**: Database and active files
2. **External Storage**: Programmed copies of databases and files
3. **External Repository**: Source code and system versions (GitHub)

**Content Backed Up**:
- Structured databases (MySQL, MongoDB)
- Multimedia and documents (Wasabi Storage)
- Source code and releases (GitHub)

---

## 8. DEPLOYMENT & INFRASTRUCTURE

### Cloud Platform
**Primary Platform**: Railway
- Hosts backend services
- Hosts microservices
- Hosts databases
- Hosts auxiliary components

### Containerization
**Technology**: Docker
**Benefits**:
- Standardize environments
- Facilitate deployment
- Enable portability
- Simplify maintenance
- Support independent microservice deployment

### Infrastructure Requirements

| Requirement | Description |
|---|---|
| **Microservice Deployment** | Each service deployable independently |
| **Environment Variables** | Secure configuration of credentials, keys, endpoints |
| **Scalability** | Ability to increase resources based on demand |
| **Monitoring** | Track availability, errors, resource consumption |
| **Environment Separation** | Development, Testing, Production |
| **Containerization** | Docker standardization |

### Development Environments

| Environment | Purpose |
|---|---|
| **Development** | Team technical work and initial feature testing |
| **Testing/Staging** | Validation before production release |
| **Production** | Real municipalities and citizens |

**Benefit**: Reduces risk of affecting production data during development

### Source Code Management
**Platform**: GitHub

**Functions**:
- Version control (track code changes)
- Collaborative development (branches and reviews)
- Modification history (track project evolution)
- Code backup (remote repository)
- Future automation (CI/CD pipeline potential)

**Repository Organization**: By independent services (gateway, claims, AI, routes, notifications, files, audit, frontend)

---

## 9. MONITORING & MAINTENANCE

### Monitored Elements

| Element | Monitoring Purpose |
|---|---|
| **Service Availability** | Detect outages or interruptions |
| **Application Errors** | Identify functional or technical failures |
| **External API Consumption** | Control costs and usage limits |
| **Work Queues** | Verify pending or failed tasks |
| **Database** | Monitor performance, growth, availability |
| **Storage** | Verify space usage and file access |
| **NATS Messaging** | Ensure inter-microservice communication |

### Maintenance Activities
- Updates and patches
- Error correction
- Security improvements
- Backup review
- Performance adjustments

---

## 10. USER ENVIRONMENTS & CLIENT INTERFACES

### Citizen Channel (WhatsApp)
**Primary Interaction Method**:
- Message to official municipality number
- System analyzes via AI
- Requests missing data if needed
- Creates claim with tracking code
- Citizen receives tracking code
- Citizen can query progress via same channel
- Notifications about status changes

**Advantages**: 
- No app download required
- Uses everyday tool
- Low adoption barrier
- Accessible to all citizens

### Complementary Web Form
**Purpose**: Secondary claim entry channel
**Characteristics**:
- Responsive design
- Works on computers, tablets, phones
- Alternative for WhatsApp non-users
- Institutional access from municipality website

### Municipal Administrative Panel
**Technology Stack**: Next.js + React + Tailwind CSS + React Query

**Access**: Web browser on computers, tablets, phones

**Core Functions**:
- View claims (filtered by status, category, priority, area, date, channel)
- Consult claim details (full view, location, evidence, conversation, history)
- Manage claim states (lifecycle transitions)
- Assign areas (route to departments)
- Validate AI suggestions (confirm/correct categorization)
- View maps (territorial claim visualization)
- Generate routes (select claims and calculate optimized routes)
- Query reports (basic management indicators)
- Administer users (create, modify, assign roles)
- Configure categories (manage categories, areas, parameters)

### Mobile Responsive Interface
**Approach**: Responsive web interface (not native app)

**Rationale**: 
- Reduces initial development complexity
- No specific app installation needed
- Enables multi-device access
- Sufficient for initial version

**Intended Users**: Agents, area managers, municipal users in field

**Capabilities**:
- View assigned claims
- Access optimized routes
- View geographic locations
- Update claim status
- Upload evidence (photos from mobile)
- Record field observations
- View action history

---

## 11. SYSTEM AVAILABILITY & CONTINUITY

### Continuity Measures

| Measure | Benefit |
|---|---|
| **Periodic Backups** | Enables recovery from data loss |
| **Retry Mechanisms in Queues** | Prevents notification or process loss |
| **Service Separation** | Module failure doesn't stop entire system |
| **Monitoring** | Early incident detection |
| **Error Logging** | Facilitates diagnosis and correction |
| **Version Control** | Enables rollback to stable versions if update fails |

### Design Philosophy
> "SmartService deberá diseñarse para brindar continuidad operativa adecuada al contexto municipal... su disponibilidad será importante para sostener la gestión de reclamos ciudadanos."

---

## 12. INTEGRATED SYSTEM DESCRIPTION

### Comprehensive Architecture Summary

**Citizen Side (Input)**:
- WhatsApp as primary channel
- Web form as secondary channel
- Minimal friction for claim entry

**Municipal Side (Processing & Management)**:
- Modern, responsive web panel
- Real-time claim visualization
- AI-assisted categorization
- Geographic mapping
- Route optimization
- Field update capability via mobile browser

**Server Side (Core Processing)**:
- Microservices architecture (NestJS)
- NATS-based asynchronous communication
- Redis/BullMQ for background tasks
- MySQL for structured data
- MongoDB for conversational/unstructured data
- Wasabi Storage for multimedia
- Docker containerization
- Railway cloud deployment

**External Integrations**:
- WhatsApp Business API (citizen channel)
- Google Maps Platform (geolocation, routing)
- AI services (classification, analysis)
- Python (specialized analysis when needed)

**Security Foundation**:
- JWT authentication
- Role-based authorization
- Complete audit trail
- Data protection mechanisms
- Backup redundancy (3-media principle)
- Regular backup schedule (daily/weekly/monthly)

**Scalability & Modularity**:
- Independent service deployment
- Decoupled architecture
- Capable of growing from single municipality to multi-municipality enterprise
- Historical data accumulation enables predictive analytics
- Future integration points for additional services

---

## KEY ARCHITECTURAL DECISIONS & RATIONALE

### 1. **Microservices Over Monolith**
- **Decision**: Microservices architecture with API Gateway
- **Rationale**: Scalability, modularity, independent evolution, reduces coupling
- **Benefit**: Can grow from municipalities to enterprises without redesign

### 2. **WhatsApp as Primary Channel**
- **Decision**: WhatsApp Business API integration
- **Rationale**: Reduces adoption barrier, uses ubiquitous citizen tool
- **Benefit**: Higher citizen participation, simpler UX

### 3. **Asynchronous Messaging (NATS)**
- **Decision**: Event-driven architecture with NATS
- **Rationale**: Decouples services, improves scalability, non-blocking
- **Benefit**: System more robust and responsive

### 4. **Dual Database Strategy**
- **Decision**: MySQL (structured) + MongoDB (unstructured)
- **Rationale**: Optimize for different data types
- **Benefit**: Balance between relational integrity and conversational flexibility

### 5. **External File Storage (Wasabi)**
- **Decision**: S3-compatible external storage
- **Rationale**: Prevents database overload, easier management
- **Benefit**: Scalable, performant file handling

### 6. **Responsive Web Instead of Native Apps**
- **Decision**: Responsive web interfaces vs. native mobile apps
- **Rationale**: Reduces complexity, avoids installation requirement
- **Benefit**: Faster time-to-market, multi-device support

### 7. **Docker + Railway Deployment**
- **Decision**: Containerized services on Railway cloud
- **Rationale**: Portability, standardization, ease of scaling
- **Benefit**: Consistent environments across development/testing/production

### 8. **GitHub for Source Control**
- **Decision**: GitHub repository with organized service structure
- **Rationale**: Version control, collaboration, CI/CD readiness
- **Benefit**: Team coordination, code backup, automation potential

---

## KEYWORDS FOUND IN DOCUMENT

✅ **arquitectura** - System architecture (microservices-based)  
✅ **microservicios** - Multiple independent services described  
✅ **NestJS** - Primary backend framework  
✅ **API Gateway** - Central entry point (not "Gateway" alone but API Gateway)  
✅ **Redis** - Work queues with BullMQ  
✅ **NATS** - Inter-service messaging system  
✅ **PostgreSQL** - ❌ NOT mentioned (MySQL used instead)  
✅ **Docker** - Containerization platform  
✅ **seguridad** - Security section with JWT, RBAC, audit  
✅ **WhatsApp Business API** - Primary citizen integration  
✅ **Google Maps Platform** - Geolocation and routing  
✅ **MongoDB** - Document database for unstructured data  
✅ **MySQL** - Relational database for structured data  
✅ **Railway** - Cloud deployment platform  
✅ **Wasabi Storage** - S3-compatible file storage  

---

## DOCUMENT STRUCTURE NOTE

The thesis follows a **business plan format** with sections on:
- Executive Summary
- Business Opportunity Description
- Technology Foundation
- **Technical Architecture** (comprehensive section extracted above)
- References

The technical architecture section is thorough and represents **complete system design** from requirements through deployment, suitable for implementation guidance.
