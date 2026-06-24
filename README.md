🦢 STOS
Swan Telegram Operating System
A multi-tenant Telegram platform built on Deno Deploy, Deno KV, Backblaze B2, and Telegram APIs.
STOS allows multiple independently owned Telegram bots to operate from a single deployment while maintaining isolated data, deterministic request processing, and platform-level security boundaries.
Overview
STOS is designed as an application platform for Telegram-based businesses.
Each bot operates within its own tenant context while sharing the same runtime infrastructure.
Use Cases
Customer support bots
Order management bots
Content publishing bots
Community management bots
Verification and onboarding workflows
Internal business automation
Features
Core Platform
Multi-tenant architecture
Tenant-isolated data storage
Deterministic request processing
Event-driven workflows
Built-in telemetry
Structured audit logging
Telegram Integration
Telegram Bot API
Telegram Gateway API
Webhook processing
Message delivery tracking
Verification workflows
Storage
Deno KV for structured data
Backblaze B2 for file storage
Partitioned storage model
Audit trail support
Security
Request validation
Signature verification
Replay protection
Permission-based access control
Tenant isolation
Architecture
Telegram APIs
      │
      ▼
Transport Layer
      │
      ▼
Security Layer
      │
      ▼
Context Resolution
      │
      ▼
Application Modules
      │
      ▼
Persistence Layer
Request Flow
Request
   │
   ▼
Validation
   │
   ▼
Authorization
   │
   ▼
Module Execution
   │
   ▼
Events
   │
   ▼
Storage
Technology Stack
Component
Technology
Runtime
Deno Deploy
Database
Deno KV
File Storage
Backblaze B2
Messaging
Telegram Bot API
Verification
Telegram Gateway API
Edge Network
Cloudflare
Project Structure
src/
├── core/
├── contracts/
├── domains/
├── adapters/
├── repositories/
└── infrastructure/

docs/
├── architecture.md
├── security.md
├── telegram.md
├── storage.md
└── operations.md
Security Model
Request Security
Telegram secret validation
Gateway signature validation
Timestamp verification
Replay protection
Access Control
Owner verification
Permission validation
Tenant-scoped repositories
Data Protection
Sensitive information is excluded from telemetry and operational logs.
Deployment
Telegram
    │
    ▼
Cloudflare
    │
    ▼
Deno Deploy
    ├── Deno KV
    └── Backblaze B2
Single deployment. Multiple isolated bot tenants.
License
Private Project