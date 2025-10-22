# SmartNeta 3-Month Phased Modernization Plan

**Timeline:** 3 months (2 months development + 1 month testing/deployment)  
**Objective:** Fix performance issues, rewrite mobile app, upgrade backend security, dockerize and deploy to Kubernetes with database read replicas and intelligent caching

---

## 1. Target Architecture Diagram (To-Be)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            KUBERNETES CLUSTER (AKS/EKS)                         │
│                                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐   │
│  │                        INGRESS CONTROLLER (NGINX)                      │   │
│  │                    SSL Termination + Rate Limiting                     │   │
│  └────────────────────┬──────────────────────────┬────────────────────────┘   │
│                       │                          │                             │
│         ┌─────────────▼───────────────┐   ┌──────▼──────────────┐             │
│         │  IONIC 7 PWA (Static)      │   │   API GATEWAY       │             │
│         │  Angular 17 + Capacitor    │   │   (Spring Cloud)    │             │
│         │  Nginx Container           │   └──────┬──────────────┘             │
│         └────────────────────────────┘          │                             │
│                                                 │                             │
│         ┌───────────────────────────────────────┴────────────────────┐        │
│         │                                                              │        │
│    ┌────▼────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│    │ CITIZEN     │  │ USER/AUTH    │  │ COMPLAINT    │  │ SURVEY          │ │
│    │ SERVICE     │  │ SERVICE      │  │ SERVICE      │  │ SERVICE         │ │
│    │             │  │              │  │              │  │                 │ │
│    │ Spring Boot │  │ Spring Boot  │  │ Spring Boot  │  │ Spring Boot     │ │
│    │ 3.2 + Java  │  │ 3.2 + Java   │  │ 3.2 + Java   │  │ 3.2 + Java      │ │
│    │ 17 LTS      │  │ 17 LTS       │  │ 17 LTS       │  │ 17 LTS          │ │
│    │             │  │              │  │              │  │                 │ │
│    │ Spring      │  │ Spring       │  │ Spring       │  │ Spring          │ │
│    │ Security    │  │ Security     │  │ Security     │  │ Security        │ │
│    │ + JWT       │  │ + JWT/OAuth2 │  │ + JWT        │  │ + JWT           │ │
│    │             │  │              │  │              │  │                 │ │
│    │ HPA: 3-10   │  │ HPA: 2-5     │  │ HPA: 2-5     │  │ HPA: 2-5        │ │
│    └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  └────────┬────────┘ │
│           │                │                  │                    │          │
│           └────────────────┴──────────────────┴────────────────────┘          │
│                                      │                                         │
│                    ┌─────────────────┴──────────────────┐                     │
│                    │                                     │                     │
│              ┌─────▼──────┐                    ┌────────▼────────┐            │
│              │   REDIS    │                    │   POSTGRESQL    │            │
│              │  CLUSTER   │◄───────────────────┤   PRIMARY       │            │
│              │            │  Cache Invalidate  │   (Read-Write)  │            │
│              │ Multi-tier │                    │                 │            │
│              │  Caching:  │                    │   - ACID Trans  │            │
│              │  - L1: App │                    │   - WAL Enabled │            │
│              │  - L2:Redis│                    │   - SSL/TLS     │            │
│              │  - L3: DB  │                    └────────┬────────┘            │
│              │            │                             │                     │
│              │  Cache TTL:│                             │ Async Replication   │
│              │  - Master: │                             │                     │
│              │    5-10min │                    ┌────────▼────────┐            │
│              │  - Dashbd: │                    │   POSTGRESQL    │            │
│              │    1-2 min │                    │   READ REPLICA  │            │
│              │  - Citizen:│                    │   (Read-Only)   │            │
│              │    30-60s  │                    │                 │            │
│              │            │                    │   - Hot Standby │            │
│              └────────────┘                    │   - Load Balance│            │
│                                                │   - Offload 60% │            │
│                                                └─────────────────┘            │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                   MONITORING & OBSERVABILITY                         │    │
│  │  Prometheus + Grafana + ELK Stack + Jaeger (Distributed Tracing)    │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘

EXTERNAL INTEGRATIONS:
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   SMS Gateway   │◄─────┤  Message Queue   │─────►│  Email Service  │
│   (Twilio/MSG91)│      │  (RabbitMQ/SQS)  │      │  (SendGrid)     │
└─────────────────┘      └──────────────────┘      └─────────────────┘

MOBILE APPS:
┌─────────────────┐      ┌──────────────────┐
│  iOS App Store  │      │  Android Play    │
│  (Capacitor)    │      │  Store (Cap.)    │
└─────────────────┘      └──────────────────┘
        │                         │
        └─────────┬───────────────┘
                  │
           ┌──────▼───────┐
           │  Ionic 7 PWA │
           │  (Offline-   │
           │   First)     │
           └──────────────┘
```

### Key Architecture Components:

1. **Mobile Layer (Ionic 7 + Angular 17)**
   - Complete rewrite with modern framework
   - Offline-first with SQLite
   - Capacitor for native features
   - Service Workers for PWA capabilities

2. **Backend Layer (Spring Boot 3.2 + Java 17)**
   - Microservices architecture (monolith first, then modular)
   - Spring Security 6+ with JWT/OAuth2
   - RESTful APIs with versioning
   - Docker containerized services

3. **Data Layer**
   - PostgreSQL 15+ Primary (Read-Write)
   - PostgreSQL Read Replica (60% read offload)
   - Redis Cluster for intelligent caching
   - Multi-tier caching strategy

4. **Infrastructure Layer**
   - Kubernetes (AKS/EKS) orchestration
   - Horizontal Pod Autoscaling (HPA)
   - Nginx Ingress with SSL/TLS
   - Monitoring with Prometheus/Grafana

---

## 2. JIRA Stories for 3-Month Delivery

### **EPIC: PHASE3M-001 - SmartNeta 3-Month Modernization**

**Epic Description:** Complete platform modernization including mobile app rewrite, backend security upgrade, dockerization, Kubernetes deployment with database read replicas and intelligent caching.

**Total Effort:** 10-11 weeks (allowing 1-2 weeks buffer within 3 months)

---

## Month 1: Foundation & Backend Modernization (Weeks 1-4)

### **Sprint 1: Backend Security & Framework Upgrades (Week 1-2)**

#### Story PHASE3M-101: Java & Spring Boot Framework Upgrade
**Priority:** Critical  
**Effort:** 5 days  
**Description:** Upgrade backend from Java 8/Spring Boot 2.0.3 to Java 17 LTS/Spring Boot 3.2.x. Includes migrating from javax.* to jakarta.* packages, updating all dependencies, fixing compilation errors, and running full test suite.

**Acceptance Criteria:**
- Java 17 LTS installed and configured
- Spring Boot upgraded to 3.2.x
- All dependencies updated to compatible versions
- Zero critical security vulnerabilities (Snyk scan)
- All existing tests passing

---

#### Story PHASE3M-102: Replace Apache Shiro with Spring Security 6
**Priority:** Critical  
**Effort:** 5 days  
**Description:** Remove Apache Shiro 1.4.0-RC2 completely and implement Spring Security 6 with JWT-based authentication. Includes OAuth2 resource server setup, custom UserDetailsService, and role-based access control (RBAC) implementation.

**Acceptance Criteria:**
- Apache Shiro dependencies removed
- Spring Security 6 configured with JWT
- Authentication endpoints working (/login, /refresh, /logout)
- Role-based authorization implemented (ADMIN, VOLUNTEER, USER)
- Password encryption with BCrypt (strength 12)
- All authentication flows tested

---

#### Story PHASE3M-103: API Security Hardening
**Priority:** Critical  
**Effort:** 3 days  
**Description:** Implement comprehensive API security including CORS configuration, rate limiting (100 req/min per user), input validation with Bean Validation, XSS/SQL injection prevention, and security headers (CSP, HSTS, X-Frame-Options).

**Acceptance Criteria:**
- CORS configured with whitelist
- Rate limiting active (Bucket4j)
- Input validation on all endpoints
- Security headers added
- OWASP ZAP scan passed

---

### **Sprint 2: Database & Caching Layer (Week 3-4)**

#### Story PHASE3M-104: PostgreSQL Migration & Setup
**Priority:** Critical  
**Effort:** 5 days  
**Description:** Migrate from MySQL to PostgreSQL 15 with encryption at rest/transit. Includes schema migration scripts (Flyway), data migration testing with 10M+ records, SSL/TLS configuration, and connection pooling optimization (HikariCP).

**Acceptance Criteria:**
- PostgreSQL 15 deployed with encryption
- All data migrated successfully
- SSL/TLS enabled for connections
- HikariCP configured (20 max pool, 5 min idle)
- Performance equal or better than MySQL

---

#### Story PHASE3M-105: Database Read Replica Configuration
**Priority:** Critical  
**Effort:** 3 days  
**Description:** Set up PostgreSQL read replica with asynchronous streaming replication. Configure read-only queries to use replica, implement connection routing logic, and monitor replication lag (target < 1 second).

**Acceptance Criteria:**
- Read replica deployed and syncing
- Replication lag < 1 second
- 60% read queries routed to replica
- Failover procedure documented
- Monitoring alerts configured

---

#### Story PHASE3M-106: Redis Caching Implementation
**Priority:** High  
**Effort:** 4 days  
**Description:** Implement Redis 7 cluster with multi-tier caching strategy: L1 (application cache), L2 (Redis), L3 (database). Configure intelligent TTLs: master data (5-10 min), dashboard aggregates (1-2 min), citizen data (30-60 sec). Implement cache invalidation on writes.

**Acceptance Criteria:**
- Redis 7 cluster deployed (3 nodes)
- Spring Cache abstraction configured
- TTL strategy implemented per data type
- Cache hit ratio > 70% for reads
- Database load reduced by 50%+

---

#### Story PHASE3M-107: Query & Index Optimization
**Priority:** High  
**Effort:** 3 days  
**Description:** Add composite indexes on citizen table (state, assembly_no, ward_no, modifieddate), optimize N+1 queries with JOIN FETCH, implement DTO projections to replace SELECT *, enforce pagination limits (max 100 records per page).

**Acceptance Criteria:**
- Composite indexes created
- Query execution time < 200ms (p95)
- N+1 queries eliminated
- Pagination enforced on all list endpoints
- Database explain plans verified

---

## Month 2: Mobile App Rewrite & Performance Optimization (Weeks 5-8)

### **Sprint 3: Mobile Foundation (Week 5-6)**

#### Story PHASE3M-201: Ionic 7 + Angular 17 Project Setup
**Priority:** Critical  
**Effort:** 3 days  
**Description:** Create new Ionic 7 project with Angular 17 and Capacitor 5. Configure TypeScript (ES2022), setup project structure with modules (auth, citizen, survey, complaint), install core dependencies (@ionic/angular, @angular/common, rxjs), and configure build pipelines.

**Acceptance Criteria:**
- Ionic 7.x project created
- Angular 17.x configured
- Capacitor 5.x for iOS/Android
- Project structure defined
- Build succeeds for dev/prod

---

#### Story PHASE3M-202: Authentication & State Management
**Priority:** Critical  
**Effort:** 5 days  
**Description:** Implement authentication module with JWT token management, secure storage (Capacitor SecureStorage), auto-refresh logic. Setup NgRx for state management with auth, citizen, and app state stores. Implement auth guards and HTTP interceptors.

**Acceptance Criteria:**
- Login/logout flows working
- JWT stored securely
- Token auto-refresh implemented
- NgRx stores configured
- Auth guards protecting routes
- HTTP interceptor adding tokens

---

#### Story PHASE3M-203: Cordova to Capacitor Plugin Migration
**Priority:** Critical  
**Effort:** 4 days  
**Description:** Migrate all Cordova plugins to Capacitor equivalents: device info, network status, camera, geolocation, filesystem, SQLite, push notifications. Test native functionality on iOS and Android devices.

**Acceptance Criteria:**
- All 10+ plugins migrated to Capacitor
- Camera, GPS, storage working
- Push notifications functional
- Tested on iOS and Android devices
- Migration guide documented

---

#### Story PHASE3M-204: Core Citizen Module Migration
**Priority:** Critical  
**Effort:** 5 days  
**Description:** Rewrite citizen search, detail view, and offline sync features. Implement optimistic UI updates, local SQLite storage for offline data (up to 10K citizens), background sync worker, and conflict resolution logic.

**Acceptance Criteria:**
- Citizen search with filters working
- Detail view with edit capability
- Offline storage (SQLite) functional
- Background sync implemented
- Conflict resolution tested

---

### **Sprint 4: Performance & Additional Modules (Week 7-8)**

#### Story PHASE3M-205: Performance Optimization - Mobile
**Priority:** High  
**Effort:** 3 days  
**Description:** Implement lazy loading for routes, virtual scrolling for large lists, image optimization (WebP format, lazy load), debounce search inputs (300ms), implement service worker for PWA caching, and optimize bundle size (target < 2MB initial load).

**Acceptance Criteria:**
- Lazy loading on all feature modules
- Virtual scrolling on lists > 100 items
- Images optimized (WebP)
- Search debounced
- Bundle size < 2MB
- Lighthouse score > 90

---

#### Story PHASE3M-206: Survey & Complaint Modules Migration
**Priority:** High  
**Effort:** 4 days  
**Description:** Migrate survey and complaint modules from old Ionic 3 app to new Ionic 7 structure. Implement form builders (reactive forms), file upload with progress, offline queue for submissions, and list views with pagination.

**Acceptance Criteria:**
- Survey creation/submission working
- Complaint lodging with attachments
- Offline queue functional
- List views with pagination
- Forms validated properly

---

#### Story PHASE3M-207: Backend Performance Optimization
**Priority:** High  
**Effort:** 3 days  
**Description:** Implement mandatory filters for citizen search endpoints (reject broad queries), optimize dashboard aggregate queries with pre-computed views, stream file uploads/downloads to object storage, configure async processing for heavy operations.

**Acceptance Criteria:**
- Mandatory filters enforced
- Dashboard queries < 500ms
- File streaming implemented
- Async jobs for reports
- p95 latency < 200ms @ 500 rps

---

#### Story PHASE3M-208: Mobile Testing Suite
**Priority:** High  
**Effort:** 3 days  
**Description:** Setup Jasmine/Karma for unit tests (target 80% coverage), configure Playwright for E2E tests, create test utilities and mocks, write tests for critical flows (auth, citizen search, survey submission).

**Acceptance Criteria:**
- Unit test coverage > 80%
- E2E tests for 5 critical flows
- CI integration (tests run on PR)
- Test documentation complete

---

## Month 3: Containerization, Kubernetes & Go-Live (Weeks 9-12)

### **Sprint 5: Dockerization & Kubernetes Setup (Week 9-10)**

#### Story PHASE3M-301: Backend Docker Containerization
**Priority:** Critical  
**Effort:** 3 days  
**Description:** Create multi-stage Dockerfiles for backend services (Maven build stage + JRE runtime), optimize image size (< 200MB), configure health checks, implement non-root user, setup Docker Compose for local development.

**Acceptance Criteria:**
- Dockerfiles for all services
- Image size < 200MB
- Health checks configured
- Non-root user (spring:spring)
- Docker Compose working locally

---

#### Story PHASE3M-302: Kubernetes Cluster Setup
**Priority:** Critical  
**Effort:** 4 days  
**Description:** Provision Kubernetes cluster (AKS/EKS), setup namespaces (dev, staging, prod), configure RBAC, setup Nginx Ingress Controller with SSL/TLS (Let's Encrypt), configure external secrets (Azure Key Vault / AWS Secrets Manager).

**Acceptance Criteria:**
- K8s cluster provisioned (3 nodes min)
- Namespaces created
- RBAC configured
- Ingress with SSL working
- Secrets externalized

---

#### Story PHASE3M-303: Service Deployments to Kubernetes
**Priority:** Critical  
**Effort:** 4 days  
**Description:** Create K8s Deployments, Services, ConfigMaps, and Secrets for all backend services. Configure resource limits/requests, liveness/readiness probes, HPA (3-10 pods for citizen service), PVCs for PostgreSQL, StatefulSet for database.

**Acceptance Criteria:**
- All services deployed
- HPA configured (3-10 pods)
- Database StatefulSet running
- Redis deployment ready
- Health checks passing

---

#### Story PHASE3M-304: CI/CD Pipeline Setup
**Priority:** Critical  
**Effort:** 4 days  
**Description:** Setup GitHub Actions workflows for backend (Maven build, Docker build/push, K8s deploy) and mobile (npm build, Docker build for PWA, deploy to CDN/K8s). Configure automated testing in CI, deployment gates, rollback procedures.

**Acceptance Criteria:**
- CI/CD pipelines for backend/mobile
- Automated tests in CI
- Docker images pushed to registry
- K8s deployments automated
- Rollback procedure tested

---

### **Sprint 6: Monitoring, Testing & Go-Live (Week 11-12)**

#### Story PHASE3M-305: Monitoring & Observability Setup
**Priority:** Critical  
**Effort:** 3 days  
**Description:** Deploy Prometheus for metrics collection, Grafana for dashboards (request rate, latency, error rate, pod health), configure alerting rules (high error rate, pod crashes, replication lag), setup ELK stack for centralized logging.

**Acceptance Criteria:**
- Prometheus scraping all services
- Grafana dashboards configured
- Alert rules active (Slack/PagerDuty)
- ELK stack collecting logs
- SLO dashboards created

---

#### Story PHASE3M-306: Load Testing & Performance Validation
**Priority:** Critical  
**Effort:** 3 days  
**Description:** Perform load testing using K6/JMeter targeting 1000 rps sustained load, validate p95 latency < 200ms, error rate < 0.5%, verify HPA scaling (3→10 pods), test database read replica load distribution, measure cache hit ratios.

**Acceptance Criteria:**
- Load test results documented
- Sustained 1000 rps achieved
- p95 latency < 200ms
- Error rate < 0.5%
- HPA scaling verified
- Cache hit ratio > 70%

---

#### Story PHASE3M-307: Security Audit & Penetration Testing
**Priority:** High  
**Effort:** 2 days  
**Description:** Run OWASP ZAP security scan on all APIs, perform penetration testing on authentication flows, verify JWT implementation, test rate limiting and CORS policies, scan container images for vulnerabilities (Trivy).

**Acceptance Criteria:**
- OWASP ZAP scan passed
- Pen testing report clean
- No critical vulnerabilities
- Rate limiting verified
- Container images scanned

---

#### Story PHASE3M-308: Production Deployment & Cutover
**Priority:** Critical  
**Effort:** 3 days  
**Description:** Execute blue-green deployment to production, migrate production database to new cluster, switch DNS/load balancer to new infrastructure, perform smoke tests, monitor system for 24 hours, rollback plan ready.

**Acceptance Criteria:**
- Production deployment successful
- Database migrated with zero downtime
- DNS switched to new cluster
- Smoke tests passed
- Monitoring active
- Rollback plan tested

---

#### Story PHASE3M-309: Documentation & Knowledge Transfer
**Priority:** High  
**Effort:** 2 days  
**Description:** Complete technical documentation (architecture diagrams, runbooks, API docs), create operational playbooks (deployment, rollback, troubleshooting), conduct knowledge transfer sessions with ops team, document monitoring alerts and responses.

**Acceptance Criteria:**
- Architecture docs complete
- Runbooks for common operations
- API documentation (Swagger/OpenAPI)
- KT sessions completed
- On-call playbooks ready

---

## 3. Summary & Timeline

### Story Summary by Priority:

| Priority | Stories | Total Effort | Percentage |
|----------|---------|-------------|------------|
| Critical | 16      | 55 days     | 73%        |
| High     | 9       | 27 days     | 27%        |
| **Total**| **25**  | **82 days** | **100%**   |

### Timeline Breakdown:

| Month | Sprints | Focus Area | Stories | Effort (days) |
|-------|---------|------------|---------|---------------|
| Month 1 | Sprint 1-2 | Backend Modernization | 7 | 28 days |
| Month 2 | Sprint 3-4 | Mobile App Rewrite | 8 | 27 days |
| Month 3 | Sprint 5-6 | K8s & Go-Live | 9 | 27 days |
| **Total** | **6 Sprints** | **Complete Platform** | **24** | **82 days** |

### Team Allocation (3 months):

| Role | Headcount | Allocation |
|------|-----------|------------|
| Backend Developer (Senior) | 1 | 100% (Month 1), 50% (Month 2-3) |
| Mobile Developer (Senior) | 2 | 20% (Month 1), 100% (Month 2), 50% (Month 3) |
| DevOps Engineer | 1 | 50% (Month 1), 50% (Month 2), 100% (Month 3) |
| QA Engineer | 1 | 30% (Month 1-2), 80% (Month 3) |
| Architect/Tech Lead | 1 | 30% across all months |

### Key Milestones:

| Milestone | Target Date | Deliverables |
|-----------|-------------|--------------|
| M1: Backend Upgraded | End of Month 1 | Java 17, Spring Security, PostgreSQL with Read Replica, Redis Caching |
| M2: Mobile App Ready | End of Month 2 | Ionic 7 app complete, tested on iOS/Android, performance optimized |
| M3: Production Ready | End of Month 3 | Dockerized, deployed to Kubernetes, monitoring active, production cutover |

### Risk Mitigation:

1. **Data Migration Risk:** Test with production-like dataset (10M+ records) in staging before prod migration
2. **Mobile App Compatibility:** Test on minimum 10 devices (iOS 13+, Android 8+) before release
3. **Performance Risk:** Load testing in staging environment 2 weeks before go-live
4. **Rollback Plan:** Blue-green deployment strategy allows instant rollback if issues detected

### Success Metrics:

| Metric | Target | Measurement |
|--------|--------|-------------|
| API Response Time (p95) | < 200ms | Prometheus |
| Mobile App Load Time | < 3 seconds | Lighthouse |
| Sustained Load | 1000 rps | Load tests |
| Error Rate | < 0.5% | Grafana |
| Availability | 99.9% | Uptime monitor |
| Database Read Offload | 60% to replica | PostgreSQL stats |
| Cache Hit Ratio | > 70% | Redis metrics |

## 4. Assumptions & Dependencies

### Assumptions:
1. Client has AWS/Azure account with appropriate quotas for Kubernetes
2. Current system documentation available
3. Test data available for staging environment
4. Access to production database for schema/data analysis
5. Apple Developer and Google Play accounts available for mobile deployment

### Dependencies:
1. Cloud infrastructure provisioning approval
2. Access to production systems for analysis
3. Availability of designated team members
4. Business stakeholder availability for UAT

### Constraints:
1. Zero downtime requirement for production cutover
2. Must maintain backward compatibility during transition
3. Mobile app must support iOS 13+ and Android 8+
4. Data privacy and GDPR compliance required

---

## Conclusion

This 3-month phased approach delivers a **production-ready, modernized SmartNeta platform** with:

✅ **Fixed performance issues** (p95 < 200ms, 1000+ rps sustained)  
✅ **Completely rewritten mobile app** (Ionic 7 + Angular 17 + Capacitor)  
✅ **Upgraded backend security** (Spring Security 6 + JWT, Java 17, Spring Boot 3.2)  
✅ **Dockerized and Kubernetes-ready** (HPA, health checks, zero-downtime deployments)  
✅ **Database scalability** (PostgreSQL with read replica, 60% read offload)  
✅ **Intelligent caching** (Redis with multi-tier strategy, >70% cache hits)  
✅ **Production monitoring** (Prometheus, Grafana, ELK stack, alerting)

