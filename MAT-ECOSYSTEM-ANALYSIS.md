# MAT Ecosystem Analysis

**Date**: 2025-12-23
**Analyzed by**: Claude Code (Opus 4.5)

## Overview

MAT (Mirai Advisory Tool) is a financial risk management platform built with microservices architecture. The system specializes in Asset Liability Management (ALM) and regulatory compliance for financial institutions.

## Microservices Inventory

| Service | Description | Spring Boot | AWS SDK v2 | Spring Cloud AWS | Status |
|---------|-------------|-------------|------------|------------------|--------|
| **param-service** | Multi-domain parametrization (scenarios, curves, behaviors, products) | 3.3.13 | 2.28.0 | 3.2.1 | **Updated** |
| **auth-service** | OAuth2/SAML authentication, token management | 3.3.13 | 2.25.31 | - | Needs sync |
| **notification-service** | Notifications, CloudWatch logs, GraphQL/WikiJS | 3.3.13 | 2.25.31 | 3.2.1 | Needs sync |
| **audit-service** | Event auditing across the platform | 3.3.13 | 2.25.31 | 3.4.0 | Needs sync |
| **admin-service** | User management, system administration | 3.3.13 | 2.25.31 | 3.4.0 | Needs sync |
| **reports-service** | Tableau integration, report generation | 3.3.13 | 2.25.31 | 3.4.0 | Needs sync |
| **athena-service** | AWS Athena/Glue operations, data extraction | 3.3.13 | 2.25.31 | 3.4.0 | Needs sync |
| **reg-adjustments-service** | Regulatory adjustments management | 3.3.13 | 2.25.31 | 3.4.0 | Needs sync |
| ~~excel-ingestor-service~~ | ~~Excel file ingestion~~ | ~~3.2.3~~ | - | - | **Deprecated** |

## Technology Stack

### Core
- **Java**: 21 (LTS)
- **Spring Boot**: 3.3.13
- **Spring Cloud**: 2023.0.3 (Leyton)
- **Build Tool**: Maven 3.x with Spring Boot Maven Plugin

### Database
- **PostgreSQL** with JPA/Hibernate 6.x
- **Hypersistence Utils** for advanced types
- **Spring Session JDBC** (auth-service)

### Security
- **Spring Security 6.x**
- **OAuth2 Authorization Server** (auth-service)
- **SAML2 Service Provider** (auth-service)
- **Jasypt** for property encryption
- **Bouncy Castle** for cryptography

### AWS Services
- **S3**: File storage with transfer manager
- **SQS**: Async messaging
- **Lambda**: Serverless functions
- **Glue**: ETL jobs
- **Athena**: Data queries
- **Step Functions**: Workflow orchestration
- **EMR**: Big data processing
- **Secrets Manager**: Credential management
- **RDS IAM Auth**: Database authentication

### Messaging
- **RabbitMQ**: Spring AMQP
- **Spring Cloud Bus**: Configuration distribution

### Observability
- **Logstash Logback Encoder**: Structured logging
- **Micrometer + Prometheus**: Metrics
- **Spring Boot Actuator**: Health/Info endpoints

### Excel Processing
- **Apache POI**: 5.5.1 (param-service), varies in other services
- **FastExcel**: 0.19.0 (param-service), varies in other services

### Internal Libraries
- **calc-core**: Financial calculations library (cannot be removed - deeply integrated)
  - Interest rate curves
  - DDA behavior modeling
  - Formula validation
  - Temporal structures

## Dependency Inconsistencies Detected

### Critical (Should Synchronize)

| Dependency | param-service | Other Services | Action |
|------------|---------------|----------------|--------|
| AWS SDK v2 | **2.28.0** | 2.25.31 | Update others |
| Logstash Encoder | **9.0** | 8.1 | Update others |
| Spring Cloud AWS | 3.2.1 | Mixed (3.2.1/3.4.0) | Standardize |

### Minor (Monitor)

| Dependency | Varies | Notes |
|------------|--------|-------|
| Lombok | 1.18.32-1.18.34 | Minor version differences |
| Apache POI | 5.2.3-5.5.1 | Consider standardizing |
| FastExcel | 0.10.12-0.19.0 | Consider standardizing |

## Recommended Agent Suite

The following agents have been created in `~/.claude/agents/`:

### 1. mat-code-reviewer.md (Existing)
Expert code reviewer for financial applications.

### 2. mat-dependency-sync.md (New)
Analyzes and synchronizes dependency versions across all microservices.

### 3. mat-aws-reviewer.md (New)
Reviews AWS configurations (S3, SQS, Lambda, Glue, Secrets Manager).

### 4. mat-security-scanner.md (New)
Audits OAuth2, SAML2, JWT tokens, and Spring Security configurations.

### 5. mat-migration-orchestrator.md (New)
Coordinates Spring Boot migrations across all services.

### 6. mat-cross-service-analyzer.md (New)
Analyzes Feign contracts, shared DTOs, and inter-service compatibility.

### 7. mat-bdd-test-generator.md (New)
Generates Cucumber/Gherkin behavior tests with mock-based patterns.

## Directory Structure

```
C:\githubMat\
├── back-admin-service/
├── back-athena-service/
├── back-audit-service/
├── back-auth-service/
├── back-config-server/
├── back-notification-service/
├── back-param-service/
├── back-reg-adjustments-service/
├── back-reports-service/
├── bigdata/
├── calc-core/
├── front-mat/
├── helm-mat-*/
└── mat-jenkins_pipelines/
```

## Next Steps

1. **Dependency Sync**: Use `mat-dependency-sync` agent to align all services
2. **Security Audit**: Run `mat-security-scanner` on auth-service first
3. **BDD Tests**: Start with param-service using `mat-bdd-test-generator`
4. **Migration Plan**: Document path to Spring Boot 3.4/3.5

---

*This document is auto-generated and should be updated after major changes.*
