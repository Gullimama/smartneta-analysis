# Minimal Critical Path to Scale Existing SmartNeta Apps

**Objective:** Support 2750 voter data downloads/second with 600K mobile users using existing tech stack (Spring Boot 2.0.3, Ionic 3.9.10, MySQL 8.0.32) without library upgrades.

**Constraints:**
- No library/framework version upgrades
- Use Docker + Kubernetes for deployment
- Can add Redis, read replicas, and reporting DB
- Minimal code changes, maximum infrastructure leverage

---

## Executive Summary

Current bottleneck: `/open/volunteer/citizens` endpoint performs unbounded SELECT on 10M+ citizen rows with minimal filters, returns full dataset as JSON, and runs on a single app instance with untuned connection pools.

**Critical Path (8 weeks):**
1. **Database layer:** Add indexes, create read replicas, offload reporting (2 weeks)
2. **Caching layer:** Redis for citizen data, master data, dashboard aggregates (2 weeks)
3. **Backend tuning:** HikariCP, Tomcat threads, query optimization, pagination enforcement (2 weeks)
4. **Containerization & K8s:** Dockerize, horizontal pod autoscaling, load balancing (2 weeks)

**Expected outcome:** 2750 req/s sustained, p95 < 500ms, zero downtime deployments.

---

## 1. Current Architecture & Bottlenecks

### 1.1 Data Flow (Voter Download)

```
Mobile App (600K users)
    ↓ (on sync/refresh)
POST /open/volunteer/citizens
    {state, assemblyNo, wardNo, date, boothNos, updatedCitizens[]}
    ↓
VolunteerController.citizens()
    → Raw SQL: SELECT * FROM citizen WHERE state=? AND assembly_no=? [AND ward_no=?] [AND modifieddate > ?]
    → No pagination, no LIMIT
    → Returns full result set as JSON (10MB avg per volunteer)
    ↓
Mobile App stores in SQLite
```

### 1.2 Critical Bottlenecks

| Layer | Issue | Impact at 2750 req/s |
|-------|-------|---------------------|
| **Database** | Missing composite indexes on (state, assembly_no, ward_no, booth_no, modifieddate) | Full table scans; p95 > 5s |
| **Database** | Single MySQL instance; all reads/writes hit primary | Connection pool saturation; query queuing |
| **Backend** | HikariCP: max-pool-size=50, idle-timeout=600s | 2750 req/s × 0.5s avg = 1375 concurrent connections needed; pool exhausted |
| **Backend** | Tomcat: default 200 threads | Thread starvation; 503 errors |
| **Backend** | No caching; every request hits DB | Repeated queries for same data (master data, recent citizens) |
| **Backend** | Unbounded queries; no enforced pagination | 10MB+ responses; heap pressure; GC pauses |
| **Mobile** | Eager sync on app start; no TTL | Thundering herd on election day mornings |
| **Deployment** | Single JAR on single VM | No horizontal scaling; single point of failure |

---

## 2. Minimal Critical Path (8 Weeks)

### Phase 1: Database Layer (Week 1-2)

#### 2.1.1 Add Composite Indexes (Day 1)

```sql
-- Primary voter download query optimization
CREATE INDEX idx_citizen_download ON citizen(state, assembly_no, ward_no, modifieddate);
CREATE INDEX idx_citizen_booth ON citizen(state, assembly_no, booth_no, modifieddate);

-- Search queries
CREATE INDEX idx_citizen_voter_id ON citizen(voter_id);
CREATE INDEX idx_citizen_mobile ON citizen(mobile);
CREATE INDEX idx_citizen_srno ON citizen(srno);

-- Dashboard aggregates
CREATE INDEX idx_citizen_status ON citizen(state, assembly_no, ward_no, booth_no, responded_status);
CREATE INDEX idx_citizen_voted ON citizen(state, assembly_no, ward_no, booth_no, voted);

-- Analyze tables
ANALYZE TABLE citizen;
ANALYZE TABLE booth;
ANALYZE TABLE ward;
ANALYZE TABLE assembly_constituency;
```

**Expected gain:** Query time 5s → 200ms (25× faster).

#### 2.1.2 MySQL Read Replicas (Day 2-3)

**Architecture:**
```
Primary MySQL (writes)
    ↓ (async replication)
Read Replica 1 (reads)
Read Replica 2 (reads)
```

**Spring Boot Configuration:**

```properties
# application.properties (add)
spring.datasource.primary.url=jdbc:mysql://mysql-primary:3306/sampark?...
spring.datasource.primary.username=root
spring.datasource.primary.password=mypass

spring.datasource.replica1.url=jdbc:mysql://mysql-replica-1:3306/sampark?...
spring.datasource.replica1.username=readonly
spring.datasource.replica1.password=readonly

spring.datasource.replica2.url=jdbc:mysql://mysql-replica-2:3306/sampark?...
spring.datasource.replica2.username=readonly
spring.datasource.replica2.password=readonly
```

**Code Changes (minimal):**

Create `DataSourceConfig.java`:

```java
@Configuration
public class DataSourceConfig {
    
    @Bean(name = "primaryDataSource")
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean(name = "replicaDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.replica1")
    public DataSource replicaDataSource() {
        // Simple round-robin between replicas can be added here
        return DataSourceBuilder.create().build();
    }
    
    @Bean(name = "primaryJdbcTemplate")
    public JdbcTemplate primaryJdbcTemplate(@Qualifier("primaryDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
    
    @Bean(name = "replicaJdbcTemplate")
    public JdbcTemplate replicaJdbcTemplate(@Qualifier("replicaDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

**Modify VolunteerController.citizens():**

```java
@Autowired
@Qualifier("replicaJdbcTemplate")
private JdbcTemplate replicaJdbcTemplate;

@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public List<Map<String, Object>> citizens(HttpServletRequest request) {
    // Use replicaJdbcTemplate instead of em.createNativeQuery()
    String sql = buildCitizenQuery(request);
    return replicaJdbcTemplate.query(sql, new CitizenRowMapper());
}
```

**Expected gain:** 3× read throughput (distribute load across 3 DB instances).

#### 2.1.3 Reporting Database (Day 4-5)

**Purpose:** Offload dashboard aggregates, analytics, CSV exports to separate DB.

**Setup:**
- Create `sampark_reporting` database
- Replicate citizen, booth, ward, assembly_constituency tables via MySQL replication or scheduled ETL
- Point dashboard/report queries to reporting DB

**Expected gain:** Isolate heavy analytical queries from transactional load.

---

### Phase 2: Caching Layer (Week 3-4)

#### 2.2.1 Redis Setup (Day 1)

**Docker Compose (dev):**

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
```

**Spring Boot Configuration:**

```properties
# application.properties (add)
spring.redis.host=redis
spring.redis.port=6379
spring.redis.timeout=2000ms
spring.cache.type=redis
spring.cache.redis.time-to-live=300000
```

**Add dependency (pom.xml):**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

#### 2.2.2 Cache Citizen Data (Day 2-4)

**Strategy:** Cache citizen data by (state, assemblyNo, wardNo, boothNos, modifiedDate).

**Code Changes:**

Enable caching in `Application.java`:

```java
@SpringBootApplication
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**Modify VolunteerController.citizens():**

```java
@Cacheable(value = "citizens", key = "#state + '_' + #assemblyNo + '_' + #wardNo + '_' + #boothNos + '_' + #date")
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public List<Map<String, Object>> citizens(
    @RequestParam String state,
    @RequestParam String assemblyNo,
    @RequestParam String wardNo,
    @RequestParam String date,
    @RequestParam String boothNos) {
    // existing logic
}
```

**TTL:** 5 minutes (300s) for citizen data; 1 hour for master data.

**Cache Invalidation:** On citizen update (POST /volunteer/citizens), evict cache:

```java
@CacheEvict(value = "citizens", key = "#state + '_' + #assemblyNo + '_' + #wardNo + '_' + #boothNos + '_' + #date")
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.POST)
@ResponseBody
public List<Map<String, Object>> uploadCitizen(...) {
    // existing logic
}
```

**Expected gain:** 80% cache hit rate → 80% of requests served from Redis (< 10ms) instead of DB (200ms).

#### 2.2.3 Cache Master Data (Day 5)

**Endpoints to cache:**
- `/volunteer/states` → TTL: 1 day
- `/volunteer/assemblyConstituencys/{stateId}` → TTL: 1 day
- `/volunteer/wardsAndBooths/{assemblyConstituencyId}` → TTL: 1 day
- `/volunteer/applicationSettings` → TTL: 1 hour
- `/volunteer/parties` → TTL: 1 day

**Code:**

```java
@Cacheable(value = "states", key = "'all'")
@RequestMapping(value = "/volunteer/states", method = RequestMethod.GET)
@ResponseBody
public HashMap<String, Object> states(...) {
    // existing logic
}

@Cacheable(value = "assemblyConstituencys", key = "#stateId")
@RequestMapping(value = "/volunteer/assemblyConstituencys/{stateId}", method = RequestMethod.GET)
@ResponseBody
public HashMap<String, Object> assemblyConstituencys(@PathVariable Long stateId) {
    // existing logic
}
```

**Expected gain:** Eliminate 50% of DB queries (master data requests).

#### 2.2.4 Cache Dashboard Aggregates (Day 6-7)

**Problem:** Dashboard queries aggregate across 10M+ rows on every request.

**Solution:** Cache aggregates for (state, assemblyNo, wardNo, boothNo) with 2-minute TTL.

**Code:**

```java
@Cacheable(value = "dashboardStats", key = "#state + '_' + #assemblyNo + '_' + #wardNo + '_' + #boothNo")
public Map<String, Object> getDashboardStats(String state, String assemblyNo, String wardNo, String boothNo) {
    // existing aggregation logic
}
```

**Expected gain:** Dashboard load time 3s → 50ms.

---

### Phase 3: Backend Tuning (Week 5-6)

#### 2.3.1 HikariCP Tuning (Day 1)

**Current config (application.properties):**
```properties
spring.datasource.hikari.maximum-pool-size=50
spring.datasource.hikari.minimum-idle=20
```

**Problem:** 2750 req/s × 0.2s avg query time = 550 concurrent connections needed per replica.

**New config:**

```properties
# Primary (writes) - conservative
spring.datasource.primary.hikari.maximum-pool-size=100
spring.datasource.primary.hikari.minimum-idle=50
spring.datasource.primary.hikari.connection-timeout=5000
spring.datasource.primary.hikari.idle-timeout=300000
spring.datasource.primary.hikari.max-lifetime=600000

# Replica (reads) - aggressive
spring.datasource.replica.hikari.maximum-pool-size=200
spring.datasource.replica.hikari.minimum-idle=100
spring.datasource.replica.hikari.connection-timeout=3000
spring.datasource.replica.hikari.idle-timeout=300000
spring.datasource.replica.hikari.max-lifetime=600000
```

**MySQL side (my.cnf):**

```ini
max_connections = 500
wait_timeout = 300
interactive_timeout = 300
```

**Expected gain:** Eliminate connection pool saturation; reduce p95 latency by 50%.

#### 2.3.2 Tomcat Thread Tuning (Day 2)

**Current:** Default 200 threads.

**New config (application.properties):**

```properties
server.tomcat.threads.max=500
server.tomcat.threads.min-spare=100
server.tomcat.accept-count=200
server.tomcat.max-connections=1000
```

**Expected gain:** Handle 2750 req/s without thread starvation.

#### 2.3.3 Query Optimization (Day 3-5)

**Problem:** `/volunteer/citizens` returns unbounded result sets.

**Solution 1: Enforce pagination**

Modify `VolunteerController.citizens()`:

```java
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public Map<String, Object> citizens(
    @RequestParam String state,
    @RequestParam String assemblyNo,
    @RequestParam String wardNo,
    @RequestParam String date,
    @RequestParam String boothNos,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "1000") int size) {
    
    // Enforce max page size
    if (size > 1000) size = 1000;
    
    String sql = buildCitizenQuery(state, assemblyNo, wardNo, date, boothNos);
    sql += " LIMIT " + size + " OFFSET " + (page * size);
    
    List<Map<String, Object>> citizens = replicaJdbcTemplate.query(sql, new CitizenRowMapper());
    
    // Return paginated response
    Map<String, Object> response = new HashMap<>();
    response.put("citizens", citizens);
    response.put("page", page);
    response.put("size", size);
    response.put("hasMore", citizens.size() == size);
    return response;
}
```

**Mobile app changes (minimal):**

Modify `IONICAPP/src/providers/localdatasync.service.ts`:

```typescript
syncCitizen() {
    let page = 0;
    let hasMore = true;
    
    const fetchPage = () => {
        this.commonService.getCitizens(this.stateName, this.assemblyNo, this.wardNo, this.last_synch_date, -1, res, page, 1000)
            .subscribe(res => {
                this.myCitizenDatabase.addUser(res.citizens).then(() => {
                    if (res.hasMore) {
                        page++;
                        fetchPage();
                    } else {
                        this.presentToast("Citizen Synced");
                    }
                });
            });
    };
    fetchPage();
}
```

**Expected gain:** Reduce response size 10MB → 1MB; reduce heap pressure; enable progressive loading.

**Solution 2: DTO Projections**

Instead of SELECT *, select only required fields:

```sql
SELECT 
    C.id, C.voter_id, C.first_name, C.family_name, C.gender, C.age, 
    C.mobile, C.srno, C.booth_no, C.ward_no, C.responded_status, C.voted, 
    C.modifieddate
FROM citizen C
WHERE ...
```

Remove heavy fields (address, latitude, longitude, party_preference) from list endpoint; fetch on detail page.

**Expected gain:** Reduce response size by 40%; reduce bandwidth and serialization time.

#### 2.3.4 Enforce Mandatory Filters (Day 6)

**Problem:** Mobile app can request all citizens in a state (10M+ rows).

**Solution:** Reject requests without assemblyNo or wardNo.

```java
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public Map<String, Object> citizens(...) {
    if (Strings.isBlank(assemblyNo) && Strings.isBlank(wardNo)) {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "assemblyNo or wardNo is required");
    }
    // existing logic
}
```

**Expected gain:** Eliminate broad scans; enforce query discipline.

#### 2.3.5 Async File Operations (Day 7)

**Problem:** `/open/getVotersDataCSV` buffers entire CSV in memory; blocks request thread.

**Solution:** Stream CSV directly to response.

Modify `MobileController.getVotersDataCSV()`:

```java
@RequestMapping(value = "/getVotersDataCSV", method = RequestMethod.GET)
public void getVotersDataCSV(HttpServletRequest request, HttpServletResponse response) throws IOException {
    String id = request.getParameter("id");
    response.setContentType("text/csv");
    response.addHeader("content-disposition", "attachment; filename=\"voters.csv\"");
    
    try (OutputStream out = response.getOutputStream()) {
        streamCitizensToCSV(id, out);
    }
}

private void streamCitizensToCSV(String id, OutputStream out) {
    // Stream query results directly to CSV without buffering
    String sql = "SELECT ... FROM citizen WHERE ...";
    replicaJdbcTemplate.query(sql, rs -> {
        // Write CSV row directly to output stream
        writeCsvRow(out, rs);
    });
}
```

**Expected gain:** Eliminate file I/O bottleneck; reduce heap usage; support concurrent downloads.

---

### Phase 4: Containerization & Kubernetes (Week 7-8)

#### 2.4.1 Dockerize Backend (Day 1-2)

**Dockerfile:**

```dockerfile
FROM openjdk:8-jre-alpine
WORKDIR /app
COPY target/smartielection-backend.jar app.jar
EXPOSE 8585

# JVM tuning for containers
ENV JAVA_OPTS="-Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+UseStringDeduplication"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**Build:**

```bash
mvn clean package -DskipTests
docker build -t smartneta-backend:latest .
```

#### 2.4.2 Kubernetes Deployment (Day 3-5)

**k8s/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smartneta-backend
spec:
  replicas: 5  # Start with 5 pods
  selector:
    matchLabels:
      app: smartneta-backend
  template:
    metadata:
      labels:
        app: smartneta-backend
    spec:
      containers:
      - name: backend
        image: smartneta-backend:latest
        ports:
        - containerPort: 8585
        env:
        - name: SPRING_DATASOURCE_PRIMARY_URL
          value: "jdbc:mysql://mysql-primary:3306/sampark?..."
        - name: SPRING_DATASOURCE_REPLICA_URL
          value: "jdbc:mysql://mysql-replica-1:3306/sampark?..."
        - name: SPRING_REDIS_HOST
          value: "redis"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8585
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8585
          initialDelaySeconds: 30
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: smartneta-backend
spec:
  type: LoadBalancer
  selector:
    app: smartneta-backend
  ports:
  - port: 80
    targetPort: 8585
```

**k8s/hpa.yaml (Horizontal Pod Autoscaler):**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: smartneta-backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: smartneta-backend
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**Expected gain:** 
- 5 pods × 500 req/s/pod = 2500 req/s baseline
- Auto-scale to 20 pods for peak load (10,000 req/s capacity)
- Zero-downtime rolling updates

#### 2.4.3 MySQL & Redis on K8s (Day 6-7)

**k8s/mysql-statefulset.yaml:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-primary
spec:
  serviceName: mysql-primary
  replicas: 1
  selector:
    matchLabels:
      app: mysql-primary
  template:
    metadata:
      labels:
        app: mysql-primary
    spec:
      containers:
      - name: mysql
        image: mysql:8.0.32
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "4Gi"
            cpu: "2000m"
          limits:
            memory: "8Gi"
            cpu: "4000m"
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 500Gi
```

**k8s/redis-deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server", "--maxmemory", "4gb", "--maxmemory-policy", "allkeys-lru"]
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "1000m"
```

#### 2.4.4 Load Testing & Tuning (Day 8-10)

**Load Test Script (JMeter/Gatling):**

```bash
# Simulate 2750 req/s for 10 minutes
ab -n 1650000 -c 2750 -t 600 -p citizen_request.json -T application/json \
   http://smartneta-backend/open/volunteer/citizens
```

**Metrics to monitor:**
- Request rate (req/s)
- Response time (p50, p95, p99)
- Error rate (%)
- CPU/memory utilization per pod
- Database connection pool usage
- Redis hit rate

**Tuning iterations:**
- Adjust HPA thresholds based on observed CPU/memory patterns
- Tune HikariCP pool sizes based on connection usage
- Adjust Redis maxmemory based on cache hit rate
- Optimize slow queries identified in MySQL slow query log

---

## 3. Mobile App Changes (Minimal)

### 3.1 Pagination Support (IONICAPP)

Modify `src/providers/common.service.ts`:

```typescript
getCitizens(state, assemblyNo, wardNo, date, boothNo, data, page = 0, size = 1000) {
  var headers = new Headers();
  headers.append("Content-Type", "application/json");
  let params = new URLSearchParams();
  params.append("state", state);
  params.append("assemblyNo", assemblyNo);
  params.append("wardNo", wardNo);
  params.append("date", date);
  params.append("boothNos", boothNo);
  params.append("page", page.toString());
  params.append("size", size.toString());

  let options_n = new RequestOptions({ headers: headers, params: params });
  return this.http
    .post(this.baseUrl + "/citizens", data, options_n)
    .map((res: Response) => res.json());
}
```

Modify `src/providers/localdatasync.service.ts`:

```typescript
syncCitizen() {
  return new Promise((resolve, reject) => {
    let page = 0;
    const fetchPage = () => {
      this.myCitizenDatabase.getDataToSync().then((localUpdates) => {
        this.commonService.getCitizens(this.stateName, this.assemblyNo, this.wardNo, 
          this.last_synch_date, -1, localUpdates, page, 1000).subscribe(res => {
          this.myCitizenDatabase.addUser(res.citizens).then(() => {
            if (res.hasMore) {
              page++;
              fetchPage();
            } else {
              this.presentToast("Citizen Synced");
              resolve(true);
            }
          });
        }, err => {
          this.presentToast("Citizen Syncing failed");
          reject(err);
        });
      });
    };
    fetchPage();
  });
}
```

### 3.2 Client-Side Request Shaping

**Add debounce to search:**

```typescript
// src/pages/search-voters/search-voters.ts
import { debounceTime } from 'rxjs/operators';

searchControl = new FormControl();

ngOnInit() {
  this.searchControl.valueChanges
    .pipe(debounceTime(500))
    .subscribe(value => {
      this.performSearch(value);
    });
}
```

**Add TTL to master data:**

```typescript
// src/providers/onetimecall.service.ts
async getStates() {
  const cached = localStorage.getItem('states');
  const cacheTime = localStorage.getItem('states_time');
  const now = Date.now();
  
  if (cached && cacheTime && (now - parseInt(cacheTime)) < 86400000) { // 24 hours
    return JSON.parse(cached);
  }
  
  const states = await this.commonService.getStates().toPromise();
  localStorage.setItem('states', JSON.stringify(states));
  localStorage.setItem('states_time', now.toString());
  return states;
}
```

---

## 4. Deployment Architecture

### 4.1 Production Architecture Diagram

```
                    ┌─────────────────┐
                    │   Load Balancer │
                    │   (K8s Ingress) │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
      ┌───────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐
      │  Backend Pod │ │ Backend  │ │  Backend   │
      │   (replica)  │ │   Pod    │ │    Pod     │
      └───────┬──────┘ └────┬─────┘ └─────┬──────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼────────┐
                    │   Redis Cache   │
                    │   (4GB memory)  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
      ┌───────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐
      │ MySQL Primary│ │  MySQL   │ │   MySQL    │
      │   (writes)   │ │ Replica 1│ │  Replica 2 │
      └──────────────┘ └──────────┘ └────────────┘
                             │
                    ┌────────▼────────┐
                    │  Reporting DB   │
                    │  (analytics)    │
                    └─────────────────┘
```

### 4.2 Resource Requirements

| Component | Instances | CPU | Memory | Storage | Cost/month (AWS) |
|-----------|-----------|-----|--------|---------|------------------|
| Backend Pods | 5-20 (auto-scale) | 1-2 vCPU | 2-4 GB | - | $200-800 |
| MySQL Primary | 1 | 4 vCPU | 16 GB | 500 GB SSD | $400 |
| MySQL Replicas | 2 | 4 vCPU | 16 GB | 500 GB SSD | $800 |
| Redis | 1 | 2 vCPU | 4 GB | - | $100 |
| Reporting DB | 1 | 2 vCPU | 8 GB | 500 GB SSD | $250 |
| Load Balancer | 1 | - | - | - | $50 |
| **Total** | | | | | **$1,800-2,400/month** |

**Note:** Costs are estimates for AWS EKS. Can be reduced 50% with reserved instances or on-premise deployment.

---

## 5. Testing & Validation

### 5.1 Load Testing Plan

**Week 7-8: Progressive load tests**

| Test | Target RPS | Duration | Success Criteria |
|------|-----------|----------|------------------|
| Baseline | 500 | 10 min | p95 < 500ms, 0% errors |
| Ramp-up | 1000 | 10 min | p95 < 500ms, < 0.1% errors |
| Target | 2750 | 10 min | p95 < 500ms, < 0.5% errors |
| Spike | 5000 | 2 min | p95 < 1s, < 1% errors |
| Endurance | 2750 | 1 hour | p95 < 500ms, < 0.5% errors |

**Tools:** Apache JMeter, Gatling, or k6.

**Metrics Dashboard:** Grafana + Prometheus
- Request rate, response time (p50/p95/p99)
- Error rate, status code distribution
- Pod CPU/memory utilization
- Database connection pool usage, query latency
- Redis hit rate, memory usage

### 5.2 Rollback Plan

**If load tests fail:**
1. Identify bottleneck via metrics (DB, cache, backend threads)
2. Tune specific layer (HikariCP, Tomcat, Redis maxmemory, HPA thresholds)
3. Re-test
4. If critical failure, rollback to previous stable version via K8s:
   ```bash
   kubectl rollout undo deployment/smartneta-backend
   ```

---

## 6. Risk Mitigation

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Database replication lag | Medium | High | Monitor lag; if > 5s, add more replicas or switch to synchronous replication for critical writes |
| Redis memory exhaustion | Medium | Medium | Set maxmemory-policy=allkeys-lru; monitor hit rate; increase memory if hit rate < 70% |
| Pod OOM kills | Low | High | Set memory limits 2× requests; enable JVM heap dumps; tune -Xmx |
| Network partition (DB) | Low | Critical | Use K8s StatefulSets with persistent volumes; enable automated failover |
| Thundering herd on election day | High | High | Pre-warm cache; implement exponential backoff in mobile app; add rate limiting (429 responses) |

---

## 7. Success Metrics

**Performance:**
- ✅ Sustained 2750 req/s for 1 hour
- ✅ p95 response time < 500ms
- ✅ p99 response time < 1s
- ✅ Error rate < 0.5%

**Scalability:**
- ✅ Auto-scale from 5 to 20 pods based on load
- ✅ Handle 5000 req/s spike for 2 minutes

**Reliability:**
- ✅ Zero downtime deployments
- ✅ Database failover < 30s
- ✅ Cache hit rate > 70%

**Cost:**
- ✅ Infrastructure cost < $2,500/month
- ✅ 50% cost reduction vs. vertical scaling single VM

---

## 8. Timeline & Milestones

| Week | Phase | Deliverables | Owner |
|------|-------|--------------|-------|
| 1-2 | Database Layer | Indexes, read replicas, reporting DB | Backend Dev + DBA |
| 3-4 | Caching Layer | Redis setup, citizen cache, master data cache, dashboard cache | Backend Dev |
| 5-6 | Backend Tuning | HikariCP, Tomcat, query optimization, pagination, file streaming | Backend Dev |
| 7-8 | K8s Deployment | Dockerize, K8s manifests, HPA, load testing, tuning | DevOps + Backend Dev |

**Total:** 8 weeks, 2 backend developers + 1 DevOps engineer.

---

## 9. Post-Launch Monitoring

**Daily (first 2 weeks):**
- Monitor Grafana dashboards for anomalies
- Review slow query log
- Check Redis hit rate
- Validate HPA scaling behavior

**Weekly:**
- Review error logs for new failure patterns
- Optimize slow queries identified in monitoring
- Tune cache TTLs based on hit rate

**Monthly:**
- Capacity planning: project growth, adjust HPA max replicas
- Cost optimization: analyze resource utilization, right-size pods
- Disaster recovery drill: test DB failover, pod restarts

---

## 10. Future Enhancements (Post-Launch)

**If additional scale needed (> 5000 req/s):**
1. **Sharding:** Partition citizen table by state or assembly_no
2. **CDN:** Cache static assets (images, templates) on CloudFront/Cloudflare
3. **Read-through cache:** Implement cache-aside pattern with automatic DB fallback
4. **Async processing:** Move heavy operations (CSV exports, bulk updates) to message queue (RabbitMQ/Kafka)
5. **Database upgrade:** Migrate to PostgreSQL with better indexing, partitioning, and connection pooling

**If library upgrades allowed:**
- Spring Boot 3.2+ (virtual threads, improved observability)
- Ionic 7+ (better performance, smaller bundle size)
- MySQL 8.0.35+ (latest performance improvements)

---

## Appendix A: Quick Reference Commands

### Build & Deploy

```bash
# Build backend
cd smartiward_fe_be_smartielection_be
mvn clean package -DskipTests

# Build Docker image
docker build -t smartneta-backend:latest .

# Push to registry
docker tag smartneta-backend:latest <registry>/smartneta-backend:latest
docker push <registry>/smartneta-backend:latest

# Deploy to K8s
kubectl apply -f k8s/mysql-statefulset.yaml
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/hpa.yaml

# Check status
kubectl get pods
kubectl get hpa
kubectl logs -f deployment/smartneta-backend
```

### Database Operations

```bash
# Connect to MySQL primary
kubectl exec -it mysql-primary-0 -- mysql -u root -p

# Check replication status
SHOW SLAVE STATUS\G

# Add indexes
mysql -u root -p sampark < add_indexes.sql

# Analyze tables
mysqlcheck -u root -p --analyze sampark
```

### Cache Operations

```bash
# Connect to Redis
kubectl exec -it redis-<pod-id> -- redis-cli

# Check cache hit rate
INFO stats

# Clear cache
FLUSHALL

# Monitor cache keys
MONITOR
```

### Load Testing

```bash
# JMeter
jmeter -n -t load_test.jmx -l results.jtl -e -o report/

# k6
k6 run --vus 2750 --duration 10m load_test.js

# Apache Bench
ab -n 165000 -c 2750 -p request.json -T application/json http://backend/open/volunteer/citizens
```

---

## Appendix B: Monitoring Queries

### Slow Queries (MySQL)

```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- Check slow queries
SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;
```

### Connection Pool Usage (HikariCP)

```sql
-- Check active connections
SHOW PROCESSLIST;

-- Check connection count by state
SELECT state, COUNT(*) FROM information_schema.processlist GROUP BY state;
```

### Redis Hit Rate

```bash
redis-cli INFO stats | grep keyspace
# Look for keyspace_hits and keyspace_misses
# Hit rate = hits / (hits + misses)
```

---

## Conclusion

This minimal critical path leverages infrastructure scaling (K8s, Redis, read replicas) and surgical backend tuning to achieve 2750 req/s with the existing tech stack. No major code rewrites or library upgrades required.

**Key success factors:**
1. **Database layer:** Indexes + read replicas eliminate query bottleneck
2. **Caching layer:** Redis offloads 70-80% of DB queries
3. **Backend tuning:** HikariCP + Tomcat tuning eliminates connection/thread saturation
4. **Horizontal scaling:** K8s HPA provides elastic capacity for peak loads

**Total effort:** 8 weeks, 3 engineers, $2,000-2,500/month infrastructure cost.

**Risk:** Low. Each phase is independently testable and reversible. No breaking changes to mobile app or admin frontend.




# JIRA Stories: Minimal Scale Path for SmartNeta

**Epic:** Scale SmartNeta to Support 2750 Voter Downloads/Second

**Timeline:** 8 weeks  
**Team:** 2 Backend Developers + 1 DevOps Engineer

---

## Epic Overview

**Epic ID:** SCALE-001  
**Epic Name:** Scale SmartNeta Platform to Support 2750 req/s with 600K Mobile Users  
**Business Value:** Enable SmartNeta to handle peak election day load (200K concurrent volunteers, 60L voter reach/day)  
**Success Criteria:**
- Sustained 2750 req/s for 1 hour
- p95 response time < 500ms
- Error rate < 0.5%
- Auto-scaling from 5 to 20 pods
- Zero downtime deployments

---

## Phase 1: Database Layer Optimization (Week 1-2)

### Story 1.1: Add Composite Indexes to Citizen Table

**Story ID:** SCALE-101  
**Story Points:** 3  
**Priority:** Critical  
**Assignee:** Backend Developer + DBA

**User Story:**
As a backend developer, I need to add composite indexes to the citizen table so that voter download queries execute in < 200ms instead of 5+ seconds.

**Acceptance Criteria:**
- [ ] Add index `idx_citizen_download` on (state, assembly_no, ward_no, modifieddate)
- [ ] Add index `idx_citizen_booth` on (state, assembly_no, booth_no, modifieddate)
- [ ] Add index `idx_citizen_voter_id` on (voter_id)
- [ ] Add index `idx_citizen_mobile` on (mobile)
- [ ] Add index `idx_citizen_srno` on (srno)
- [ ] Add index `idx_citizen_status` on (state, assembly_no, ward_no, booth_no, responded_status)
- [ ] Add index `idx_citizen_voted` on (state, assembly_no, ward_no, booth_no, voted)
- [ ] Run ANALYZE TABLE on citizen, booth, ward, assembly_constituency
- [ ] Verify query execution plan uses new indexes (EXPLAIN)
- [ ] Measure query time before/after (document in comment)
- [ ] Test on staging with 10M+ citizen records

**Technical Details:**
```sql
-- Execute on production during low-traffic window
CREATE INDEX idx_citizen_download ON citizen(state, assembly_no, ward_no, modifieddate);
CREATE INDEX idx_citizen_booth ON citizen(state, assembly_no, booth_no, modifieddate);
CREATE INDEX idx_citizen_voter_id ON citizen(voter_id);
CREATE INDEX idx_citizen_mobile ON citizen(mobile);
CREATE INDEX idx_citizen_srno ON citizen(srno);
CREATE INDEX idx_citizen_status ON citizen(state, assembly_no, ward_no, booth_no, responded_status);
CREATE INDEX idx_citizen_voted ON citizen(state, assembly_no, ward_no, booth_no, voted);

ANALYZE TABLE citizen;
ANALYZE TABLE booth;
ANALYZE TABLE ward;
ANALYZE TABLE assembly_constituency;
```

**Testing:**
```sql
-- Before/after comparison
EXPLAIN SELECT * FROM citizen 
WHERE state = 'Karnataka' AND assembly_no = '123' AND ward_no = '5' 
AND modifieddate > '2025-01-01';
```

**Definition of Done:**
- Indexes created on production
- Query time reduced from 5s to < 200ms (documented)
- No degradation in write performance
- Code review approved
- Staging tested with 10M records

---

### Story 1.2: Set Up MySQL Read Replicas

**Story ID:** SCALE-102  
**Story Points:** 8  
**Priority:** Critical  
**Assignee:** DevOps Engineer + Backend Developer

**User Story:**
As a DevOps engineer, I need to set up 2 MySQL read replicas so that read queries are distributed across 3 database instances, increasing read throughput by 3×.

**Acceptance Criteria:**
- [ ] Provision 2 MySQL 8.0.32 read replica instances (same specs as primary)
- [ ] Configure async replication from primary to replicas
- [ ] Verify replication lag < 1 second
- [ ] Create readonly database user for replicas
- [ ] Configure connection pooling for replicas
- [ ] Set up monitoring for replication lag (Prometheus/Grafana)
- [ ] Document failover procedure
- [ ] Test replica promotion to primary (disaster recovery drill)

**Technical Details:**

**Infrastructure:**
- Primary: 4 vCPU, 16 GB RAM, 500 GB SSD
- Replica 1: 4 vCPU, 16 GB RAM, 500 GB SSD
- Replica 2: 4 vCPU, 16 GB RAM, 500 GB SSD

**Replication Setup (on Primary):**
```sql
-- Create replication user
CREATE USER 'replicator'@'%' IDENTIFIED BY 'secure_password';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;

-- Get binary log position
SHOW MASTER STATUS;
```

**Replication Setup (on Replicas):**
```sql
CHANGE MASTER TO
  MASTER_HOST='mysql-primary',
  MASTER_USER='replicator',
  MASTER_PASSWORD='secure_password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=12345;

START SLAVE;
SHOW SLAVE STATUS\G
```

**Monitoring:**
```sql
-- Check replication lag
SHOW SLAVE STATUS\G
-- Look for Seconds_Behind_Master (should be < 1)
```

**Definition of Done:**
- 2 replicas running and replicating
- Replication lag < 1 second (verified over 24 hours)
- Monitoring alerts configured for lag > 5 seconds
- Failover procedure documented and tested
- Code review approved

---

### Story 1.3: Implement Multi-DataSource Configuration in Spring Boot

**Story ID:** SCALE-103  
**Story Points:** 5  
**Priority:** Critical  
**Assignee:** Backend Developer  
**Dependencies:** SCALE-102

**User Story:**
As a backend developer, I need to configure Spring Boot to use read replicas for read queries and primary for write queries, so that read load is distributed across 3 database instances.

**Acceptance Criteria:**
- [ ] Create `DataSourceConfig.java` with primary and replica data sources
- [ ] Configure HikariCP connection pools for each data source
- [ ] Create `primaryJdbcTemplate` and `replicaJdbcTemplate` beans
- [ ] Update `VolunteerController.citizens()` to use `replicaJdbcTemplate`
- [ ] Update all read-only endpoints to use replica data source
- [ ] Ensure all write operations use primary data source
- [ ] Add connection pool metrics to Actuator
- [ ] Test on staging with load test (500 req/s)
- [ ] Verify read queries hit replicas (check DB connection logs)

**Technical Details:**

**application.properties:**
```properties
# Primary (writes)
spring.datasource.primary.url=jdbc:mysql://mysql-primary:3306/sampark?...
spring.datasource.primary.username=root
spring.datasource.primary.password=${DB_PRIMARY_PASSWORD}
spring.datasource.primary.hikari.maximum-pool-size=100
spring.datasource.primary.hikari.minimum-idle=50

# Replica (reads)
spring.datasource.replica.url=jdbc:mysql://mysql-replica-1:3306/sampark?...
spring.datasource.replica.username=readonly
spring.datasource.replica.password=${DB_REPLICA_PASSWORD}
spring.datasource.replica.hikari.maximum-pool-size=200
spring.datasource.replica.hikari.minimum-idle=100
```

**DataSourceConfig.java:**
```java
@Configuration
public class DataSourceConfig {
    
    @Bean(name = "primaryDataSource")
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean(name = "replicaDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean(name = "primaryJdbcTemplate")
    public JdbcTemplate primaryJdbcTemplate(@Qualifier("primaryDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
    
    @Bean(name = "replicaJdbcTemplate")
    public JdbcTemplate replicaJdbcTemplate(@Qualifier("replicaDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

**VolunteerController.java (update):**
```java
@Autowired
@Qualifier("replicaJdbcTemplate")
private JdbcTemplate replicaJdbcTemplate;

@Autowired
@Qualifier("primaryJdbcTemplate")
private JdbcTemplate primaryJdbcTemplate;

@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public List<Map<String, Object>> citizens(HttpServletRequest request) {
    // Use replica for reads
    String sql = buildCitizenQuery(request);
    return replicaJdbcTemplate.query(sql, new CitizenRowMapper());
}

@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.POST)
@ResponseBody
public List<Map<String, Object>> uploadCitizen(...) {
    // Use primary for writes
    for (Citizen citizen : citizens) {
        primaryJdbcTemplate.update("UPDATE citizen SET ...", params);
    }
    return citizens(request);
}
```

**Testing:**
- Load test with 500 req/s for 5 minutes
- Verify replica connection pool usage in Actuator metrics
- Check MySQL processlist on primary vs replicas

**Definition of Done:**
- Multi-datasource configuration implemented
- All read queries use replica
- All write queries use primary
- Load test passes (500 req/s, p95 < 500ms)
- Code review approved
- Merged to main branch

---

### Story 1.4: Set Up Reporting Database

**Story ID:** SCALE-104  
**Story Points:** 5  
**Priority:** Medium  
**Assignee:** DevOps Engineer + Backend Developer

**User Story:**
As a DevOps engineer, I need to set up a separate reporting database so that heavy analytical queries (dashboard aggregates, CSV exports) don't impact transactional load.

**Acceptance Criteria:**
- [ ] Provision MySQL reporting database instance (2 vCPU, 8 GB RAM, 500 GB SSD)
- [ ] Set up replication from primary to reporting DB
- [ ] Create readonly user for reporting DB
- [ ] Configure Spring Boot data source for reporting DB
- [ ] Migrate dashboard aggregate queries to reporting DB
- [ ] Migrate CSV export queries to reporting DB
- [ ] Test dashboard and CSV export functionality
- [ ] Monitor reporting DB query performance

**Technical Details:**

**application.properties:**
```properties
# Reporting DB (analytics)
spring.datasource.reporting.url=jdbc:mysql://mysql-reporting:3306/sampark?...
spring.datasource.reporting.username=readonly
spring.datasource.reporting.password=${DB_REPORTING_PASSWORD}
spring.datasource.reporting.hikari.maximum-pool-size=50
spring.datasource.reporting.hikari.minimum-idle=20
```

**DataSourceConfig.java (add):**
```java
@Bean(name = "reportingDataSource")
@ConfigurationProperties(prefix = "spring.datasource.reporting")
public DataSource reportingDataSource() {
    return DataSourceBuilder.create().build();
}

@Bean(name = "reportingJdbcTemplate")
public JdbcTemplate reportingJdbcTemplate(@Qualifier("reportingDataSource") DataSource ds) {
    return new JdbcTemplate(ds);
}
```

**CommonController.java (update dashboard queries):**
```java
@Autowired
@Qualifier("reportingJdbcTemplate")
private JdbcTemplate reportingJdbcTemplate;

@RequestMapping(value = "/api/dashboardStats", method = RequestMethod.GET)
@ResponseBody
public Map<String, Object> getDashboardStats(...) {
    // Use reporting DB for aggregates
    String sql = "SELECT COUNT(*), responded_status FROM citizen WHERE ...";
    return reportingJdbcTemplate.queryForMap(sql);
}
```

**Definition of Done:**
- Reporting DB provisioned and replicating
- Dashboard queries migrated to reporting DB
- CSV export queries migrated to reporting DB
- Functional testing passed
- No impact on transactional load (verified via monitoring)
- Code review approved

---

## Phase 2: Caching Layer (Week 3-4)

### Story 2.1: Set Up Redis Cache Infrastructure

**Story ID:** SCALE-201  
**Story Points:** 3  
**Priority:** Critical  
**Assignee:** DevOps Engineer

**User Story:**
As a DevOps engineer, I need to set up a Redis cache instance so that frequently accessed data can be cached in memory, reducing database load by 70-80%.

**Acceptance Criteria:**
- [ ] Provision Redis 7 instance (2 vCPU, 4 GB RAM)
- [ ] Configure Redis with maxmemory=4gb and maxmemory-policy=allkeys-lru
- [ ] Set up Redis monitoring (Prometheus/Grafana)
- [ ] Configure Redis persistence (RDB snapshots)
- [ ] Test Redis failover and recovery
- [ ] Document Redis operations (flush, monitor, backup)

**Technical Details:**

**Docker Compose (dev):**
```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 4gb --maxmemory-policy allkeys-lru --save 900 1 --save 300 10
    volumes:
      - redis-data:/data
    restart: unless-stopped

volumes:
  redis-data:
```

**Kubernetes (prod):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: 
          - redis-server
          - --maxmemory
          - "4gb"
          - --maxmemory-policy
          - allkeys-lru
          - --save
          - "900"
          - "1"
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "1000m"
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc
```

**Monitoring:**
- Redis hit rate: `INFO stats | grep keyspace`
- Memory usage: `INFO memory`
- Connected clients: `INFO clients`

**Definition of Done:**
- Redis instance running in dev and staging
- Monitoring configured
- Failover tested
- Operations documented
- Ready for application integration

---

### Story 2.2: Integrate Spring Boot with Redis Cache

**Story ID:** SCALE-202  
**Story Points:** 3  
**Priority:** Critical  
**Assignee:** Backend Developer  
**Dependencies:** SCALE-201

**User Story:**
As a backend developer, I need to integrate Spring Boot with Redis so that I can use `@Cacheable` annotations to cache API responses.

**Acceptance Criteria:**
- [ ] Add `spring-boot-starter-data-redis` dependency to pom.xml
- [ ] Configure Redis connection in application.properties
- [ ] Enable caching with `@EnableCaching` annotation
- [ ] Configure cache TTLs for different cache regions
- [ ] Add cache metrics to Actuator
- [ ] Test cache operations (get, set, evict)
- [ ] Verify cache hit/miss in Redis CLI

**Technical Details:**

**pom.xml:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**application.properties:**
```properties
# Redis configuration
spring.redis.host=redis
spring.redis.port=6379
spring.redis.timeout=2000ms
spring.redis.jedis.pool.max-active=50
spring.redis.jedis.pool.max-idle=20
spring.redis.jedis.pool.min-idle=10

# Cache configuration
spring.cache.type=redis
spring.cache.redis.time-to-live=300000
spring.cache.redis.cache-null-values=false
spring.cache.redis.use-key-prefix=true
```

**CacheConfig.java:**
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .disableCachingNullValues()
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // Citizen data: 5 minutes
        cacheConfigurations.put("citizens", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(5)));
        
        // Master data: 1 day
        cacheConfigurations.put("states", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofDays(1)));
        cacheConfigurations.put("assemblyConstituencys", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofDays(1)));
        cacheConfigurations.put("wards", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofDays(1)));
        
        // Dashboard: 2 minutes
        cacheConfigurations.put("dashboardStats", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(2)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(cacheConfiguration())
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }
}
```

**Application.java:**
```java
@SpringBootApplication
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**Testing:**
```bash
# Connect to Redis
redis-cli

# Monitor cache operations
MONITOR

# Check keys
KEYS *

# Check cache hit rate
INFO stats
```

**Definition of Done:**
- Redis integration complete
- Cache configuration tested
- Cache metrics available in Actuator
- Verified cache operations in Redis CLI
- Code review approved

---

### Story 2.3: Implement Citizen Data Caching

**Story ID:** SCALE-203  
**Story Points:** 5  
**Priority:** Critical  
**Assignee:** Backend Developer  
**Dependencies:** SCALE-202

**User Story:**
As a backend developer, I need to cache citizen data responses so that 80% of citizen download requests are served from Redis cache (< 10ms) instead of database (200ms).

**Acceptance Criteria:**
- [ ] Add `@Cacheable` to `VolunteerController.citizens()` method
- [ ] Implement cache key strategy based on (state, assemblyNo, wardNo, boothNos, date)
- [ ] Add `@CacheEvict` to citizen update methods
- [ ] Test cache hit/miss behavior
- [ ] Measure cache hit rate (target > 70%)
- [ ] Verify response time improvement (200ms → 10ms for cache hits)
- [ ] Load test with 1000 req/s

**Technical Details:**

**VolunteerController.java:**
```java
@Cacheable(
    value = "citizens", 
    key = "#state + '_' + #assemblyNo + '_' + #wardNo + '_' + #boothNos + '_' + #date",
    unless = "#result == null || #result.isEmpty()"
)
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public List<Map<String, Object>> citizens(
    @RequestParam String state,
    @RequestParam String assemblyNo,
    @RequestParam String wardNo,
    @RequestParam String date,
    @RequestParam String boothNos) {
    
    log.info("Cache miss - fetching from DB: state={}, assemblyNo={}, wardNo={}", 
        state, assemblyNo, wardNo);
    
    // Existing DB query logic
    String sql = buildCitizenQuery(state, assemblyNo, wardNo, date, boothNos);
    return replicaJdbcTemplate.query(sql, new CitizenRowMapper());
}

@CacheEvict(
    value = "citizens", 
    key = "#request.getParameter('state') + '_' + #request.getParameter('assemblyNo') + '_' + " +
          "#request.getParameter('wardNo') + '_' + #request.getParameter('boothNos') + '_' + " +
          "#request.getParameter('date')"
)
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.POST)
@ResponseBody
public List<Map<String, Object>> uploadCitizen(
    HttpServletRequest request,
    @RequestBody ArrayList<Citizen> citizens) {
    
    log.info("Cache evicted - citizen data updated");
    
    // Existing update logic
    for (Citizen citizen : citizens) {
        // Update citizen
    }
    
    return citizens(request);
}
```

**Testing Script:**
```bash
# Test cache hit
curl "http://localhost:8585/open/volunteer/citizens?state=Karnataka&assemblyNo=123&wardNo=5&date=-1&boothNos=-1"

# Check Redis
redis-cli
KEYS citizens::*
GET "citizens::Karnataka_123_5_-1_-1"

# Measure response time
ab -n 1000 -c 100 "http://localhost:8585/open/volunteer/citizens?state=Karnataka&assemblyNo=123&wardNo=5&date=-1&boothNos=-1"
```

**Metrics to Track:**
- Cache hit rate: `INFO stats` (keyspace_hits / (keyspace_hits + keyspace_misses))
- Response time: p50, p95, p99
- Database query count (should drop by 70-80%)

**Definition of Done:**
- Caching implemented and tested
- Cache hit rate > 70% (measured over 1 hour)
- Response time for cache hits < 20ms
- Load test passes (1000 req/s, p95 < 100ms)
- Code review approved

---

### Story 2.4: Implement Master Data Caching

**Story ID:** SCALE-204  
**Story Points:** 3  
**Priority:** High  
**Assignee:** Backend Developer  
**Dependencies:** SCALE-202

**User Story:**
As a backend developer, I need to cache master data (states, assemblies, wards, booths, parties) so that these frequently accessed, rarely changing datasets are served from cache, eliminating 50% of database queries.

**Acceptance Criteria:**
- [ ] Add `@Cacheable` to all master data endpoints
- [ ] Configure 1-day TTL for master data caches
- [ ] Add `@CacheEvict` to master data update endpoints (admin)
- [ ] Test cache behavior for all master data endpoints
- [ ] Verify cache hit rate > 90% for master data
- [ ] Document cache invalidation process for admins

**Technical Details:**

**VolunteerController.java:**
```java
@Cacheable(value = "states", key = "'all'")
@RequestMapping(value = "/volunteer/states", method = RequestMethod.GET)
@ResponseBody
public HashMap<String, Object> states(HttpServletRequest request, HttpServletResponse response) {
    log.info("Cache miss - fetching states from DB");
    HashMap<String, Object> result = new HashMap<>();
    result.put("states", stateAssemblyRepository.findAll());
    return result;
}

@Cacheable(value = "assemblyConstituencys", key = "#stateId")
@RequestMapping(value = "/volunteer/assemblyConstituencys/{stateId}", method = RequestMethod.GET)
@ResponseBody
public HashMap<String, Object> assemblyConstituencys(
    HttpServletRequest request, 
    HttpServletResponse response,
    @PathVariable("stateId") Long stateId) {
    
    log.info("Cache miss - fetching assemblies for state: {}", stateId);
    HashMap<String, Object> result = new HashMap<>();
    result.put("assemblyConstituencys", acRepository.findByState(stateId));
    return result;
}

@Cacheable(value = "wardsAndBooths", key = "#assemblyConstituencyId")
@RequestMapping(value = "/volunteer/wardsAndBooths/{assemblyConstituencyId}", method = RequestMethod.GET)
@ResponseBody
public HashMap<String, Object> wards(
    HttpServletRequest request, 
    HttpServletResponse response,
    @PathVariable("assemblyConstituencyId") Long assemblyConstituencyId) {
    
    log.info("Cache miss - fetching wards/booths for assembly: {}", assemblyConstituencyId);
    HashMap<String, Object> result = new HashMap<>();
    result.put("wards", wardRepository.findAllByAssemblyConstituencyId(assemblyConstituencyId));
    result.put("booths", boothRepository.findAllByAssemblyConstituencyId(assemblyConstituencyId));
    return result;
}

@Cacheable(value = "parties", key = "'all'")
@RequestMapping(value = "/volunteer/parties", method = RequestMethod.GET)
@ResponseBody
public HashMap<String, Object> parties() {
    log.info("Cache miss - fetching parties from DB");
    HashMap<String, Object> result = new HashMap<>();
    result.put("parties", partyService.findAll());
    return result;
}

@Cacheable(value = "applicationSettings", key = "'all'")
@RequestMapping(value = "/volunteer/applicationSettings", method = RequestMethod.GET)
@ResponseBody
public HashMap<String, Object> applicationSettings() {
    log.info("Cache miss - fetching application settings from DB");
    HashMap<String, Object> result = new HashMap<>();
    result.put("settings", applicationSettingsRepository.findAll());
    return result;
}
```

**Cache Invalidation (Admin Controller):**
```java
@CacheEvict(value = "states", allEntries = true)
@RequestMapping(value = "/api/state", method = RequestMethod.POST)
@ResponseBody
public ResponseWrapper createState(@RequestBody StateAssembly state) {
    log.info("Cache evicted - state created/updated");
    // Existing logic
}

@CacheEvict(value = "assemblyConstituencys", allEntries = true)
@RequestMapping(value = "/api/assemblyConstituency", method = RequestMethod.POST)
@ResponseBody
public ResponseWrapper createAssembly(@RequestBody AssemblyConstituency ac) {
    log.info("Cache evicted - assembly created/updated");
    // Existing logic
}
```

**Testing:**
```bash
# Test cache for each endpoint
curl http://localhost:8585/open/volunteer/states
curl http://localhost:8585/open/volunteer/assemblyConstituencys/1
curl http://localhost:8585/open/volunteer/wardsAndBooths/123
curl http://localhost:8585/open/volunteer/parties

# Verify in Redis
redis-cli
KEYS states::*
KEYS assemblyConstituencys::*
KEYS wardsAndBooths::*
KEYS parties::*
```

**Definition of Done:**
- All master data endpoints cached
- Cache hit rate > 90% for master data (measured over 24 hours)
- Cache invalidation tested
- Documentation updated
- Code review approved

---

### Story 2.5: Implement Dashboard Aggregate Caching

**Story ID:** SCALE-205  
**Story Points:** 5  
**Priority:** High  
**Assignee:** Backend Developer  
**Dependencies:** SCALE-202

**User Story:**
As a backend developer, I need to cache dashboard aggregate statistics so that dashboard load time is reduced from 3 seconds to < 100ms.

**Acceptance Criteria:**
- [ ] Identify all dashboard aggregate queries
- [ ] Add `@Cacheable` to dashboard stat methods
- [ ] Configure 2-minute TTL for dashboard cache
- [ ] Implement cache warming for common dashboard queries
- [ ] Test dashboard performance (target < 100ms)
- [ ] Verify cache hit rate > 80%
- [ ] Load test dashboard endpoint (500 req/s)

**Technical Details:**

**DashboardService.java (create):**
```java
@Service
public class DashboardService {
    
    @Autowired
    @Qualifier("reportingJdbcTemplate")
    private JdbcTemplate reportingJdbcTemplate;
    
    @Cacheable(
        value = "dashboardStats",
        key = "#state + '_' + #assemblyNo + '_' + #wardNo + '_' + #boothNo"
    )
    public Map<String, Object> getDashboardStats(
        String state, 
        String assemblyNo, 
        String wardNo, 
        String boothNo) {
        
        log.info("Cache miss - computing dashboard stats: state={}, assemblyNo={}, wardNo={}, boothNo={}", 
            state, assemblyNo, wardNo, boothNo);
        
        Map<String, Object> stats = new HashMap<>();
        
        // Total voters
        String totalSql = "SELECT COUNT(*) FROM citizen WHERE state = ? AND assembly_no = ?";
        List<Object> params = new ArrayList<>();
        params.add(state);
        params.add(assemblyNo);
        
        if (!"-1".equals(wardNo)) {
            totalSql += " AND ward_no = ?";
            params.add(wardNo);
        }
        if (!"-1".equals(boothNo)) {
            totalSql += " AND booth_no = ?";
            params.add(boothNo);
        }
        
        Integer total = reportingJdbcTemplate.queryForObject(totalSql, params.toArray(), Integer.class);
        stats.put("totalVoters", total);
        
        // Responded status breakdown
        String statusSql = "SELECT responded_status, COUNT(*) as count FROM citizen " +
                          "WHERE state = ? AND assembly_no = ?";
        if (!"-1".equals(wardNo)) {
            statusSql += " AND ward_no = ?";
        }
        if (!"-1".equals(boothNo)) {
            statusSql += " AND booth_no = ?";
        }
        statusSql += " GROUP BY responded_status";
        
        List<Map<String, Object>> statusBreakdown = reportingJdbcTemplate.queryForList(statusSql, params.toArray());
        stats.put("statusBreakdown", statusBreakdown);
        
        // Voted count
        String votedSql = "SELECT COUNT(*) FROM citizen WHERE voted = 1 AND state = ? AND assembly_no = ?";
        if (!"-1".equals(wardNo)) {
            votedSql += " AND ward_no = ?";
        }
        if (!"-1".equals(boothNo)) {
            votedSql += " AND booth_no = ?";
        }
        
        Integer voted = reportingJdbcTemplate.queryForObject(votedSql, params.toArray(), Integer.class);
        stats.put("votedCount", voted);
        
        return stats;
    }
    
    @CacheEvict(value = "dashboardStats", allEntries = true)
    public void evictDashboardCache() {
        log.info("Dashboard cache evicted");
    }
}
```

**CommonController.java (update):**
```java
@Autowired
private DashboardService dashboardService;

@RequestMapping(value = "/api/dashboardStats", method = RequestMethod.GET)
@ResponseBody
public ResponseWrapper getDashboardStats(
    @RequestParam String state,
    @RequestParam String assemblyNo,
    @RequestParam(defaultValue = "-1") String wardNo,
    @RequestParam(defaultValue = "-1") String boothNo) {
    
    Map<String, Object> stats = dashboardService.getDashboardStats(state, assemblyNo, wardNo, boothNo);
    return new ResponseWrapper().asData(stats);
}
```

**Cache Warming (optional - scheduled task):**
```java
@Component
public class CacheWarmingScheduler {
    
    @Autowired
    private DashboardService dashboardService;
    
    @Autowired
    private StateAssemblyRepository stateAssemblyRepository;
    
    @Scheduled(fixedRate = 120000) // Every 2 minutes
    public void warmDashboardCache() {
        log.info("Warming dashboard cache...");
        
        // Warm cache for all states and top assemblies
        List<StateAssembly> states = stateAssemblyRepository.findAll();
        for (StateAssembly state : states) {
            dashboardService.getDashboardStats(state.getName(), "123", "-1", "-1");
        }
    }
}
```

**Testing:**
```bash
# Test dashboard endpoint
time curl "http://localhost:8585/api/dashboardStats?state=Karnataka&assemblyNo=123&wardNo=5&boothNo=-1"

# First call: ~3s (cache miss)
# Second call: ~50ms (cache hit)

# Load test
ab -n 5000 -c 500 "http://localhost:8585/api/dashboardStats?state=Karnataka&assemblyNo=123&wardNo=5&boothNo=-1"
```

**Definition of Done:**
- Dashboard caching implemented
- Dashboard load time < 100ms (cache hit)
- Cache hit rate > 80% (measured over 1 hour)
- Load test passes (500 req/s, p95 < 200ms)
- Code review approved

---

## Phase 3: Backend Tuning (Week 5-6)

### Story 3.1: Tune HikariCP Connection Pools

**Story ID:** SCALE-301  
**Story Points:** 3  
**Priority:** Critical  
**Assignee:** Backend Developer

**User Story:**
As a backend developer, I need to tune HikariCP connection pool settings so that the application can handle 2750 concurrent requests without connection pool saturation.

**Acceptance Criteria:**
- [ ] Calculate optimal pool sizes based on load (2750 req/s × 0.2s = 550 connections)
- [ ] Update application.properties with tuned HikariCP settings
- [ ] Configure separate pools for primary, replica, and reporting data sources
- [ ] Update MySQL max_connections to support new pool sizes
- [ ] Add connection pool metrics to Actuator
- [ ] Load test with 2000 req/s
- [ ] Verify no connection pool exhaustion errors

**Technical Details:**

**application.properties:**
```properties
# Primary (writes) - conservative
spring.datasource.primary.hikari.maximum-pool-size=100
spring.datasource.primary.hikari.minimum-idle=50
spring.datasource.primary.hikari.connection-timeout=5000
spring.datasource.primary.hikari.idle-timeout=300000
spring.datasource.primary.hikari.max-lifetime=600000
spring.datasource.primary.hikari.validation-timeout=3000
spring.datasource.primary.hikari.leak-detection-threshold=60000

# Replica (reads) - aggressive
spring.datasource.replica.hikari.maximum-pool-size=200
spring.datasource.replica.hikari.minimum-idle=100
spring.datasource.replica.hikari.connection-timeout=3000
spring.datasource.replica.hikari.idle-timeout=300000
spring.datasource.replica.hikari.max-lifetime=600000
spring.datasource.replica.hikari.validation-timeout=3000
spring.datasource.replica.hikari.leak-detection-threshold=60000

# Reporting (analytics) - moderate
spring.datasource.reporting.hikari.maximum-pool-size=50
spring.datasource.reporting.hikari.minimum-idle=20
spring.datasource.reporting.hikari.connection-timeout=10000
spring.datasource.reporting.hikari.idle-timeout=600000
spring.datasource.reporting.hikari.max-lifetime=900000
```

**MySQL Configuration (my.cnf):**
```ini
[mysqld]
max_connections = 500
wait_timeout = 300
interactive_timeout = 300
max_allowed_packet = 64M
```

**Monitoring Queries:**
```sql
-- Check current connections
SHOW PROCESSLIST;

-- Check connection count by state
SELECT state, COUNT(*) 
FROM information_schema.processlist 
GROUP BY state;

-- Check max connections
SHOW VARIABLES LIKE 'max_connections';
```

**Actuator Metrics:**
```bash
# Check connection pool metrics
curl http://localhost:8585/actuator/metrics/hikaricp.connections.active
curl http://localhost:8585/actuator/metrics/hikaricp.connections.idle
curl http://localhost:8585/actuator/metrics/hikaricp.connections.max
curl http://localhost:8585/actuator/metrics/hikaricp.connections.pending
```

**Load Test:**
```bash
# Test with 2000 req/s for 5 minutes
ab -n 600000 -c 2000 -t 300 "http://localhost:8585/open/volunteer/citizens?..."

# Monitor connection pool usage during test
watch -n 1 'curl -s http://localhost:8585/actuator/metrics/hikaricp.connections.active | jq'
```

**Definition of Done:**
- HikariCP settings tuned
- MySQL max_connections updated
- Load test passes (2000 req/s, no connection errors)
- Connection pool metrics available in Actuator
- Grafana dashboard created for connection pool monitoring
- Code review approved

---

### Story 3.2: Tune Tomcat Thread Pool

**Story ID:** SCALE-302  
**Story Points:** 2  
**Priority:** Critical  
**Assignee:** Backend Developer

**User Story:**
As a backend developer, I need to tune Tomcat thread pool settings so that the application can handle 2750 concurrent requests without thread starvation.

**Acceptance Criteria:**
- [ ] Calculate optimal thread count (2750 req/s × 0.2s = 550 threads minimum)
- [ ] Update application.properties with tuned Tomcat settings
- [ ] Add thread pool metrics to Actuator
- [ ] Load test with 2000 req/s
- [ ] Verify no thread pool exhaustion (503 errors)
- [ ] Monitor thread pool usage in Grafana

**Technical Details:**

**application.properties:**
```properties
# Tomcat thread pool
server.tomcat.threads.max=500
server.tomcat.threads.min-spare=100
server.tomcat.accept-count=200
server.tomcat.max-connections=1000

# Connection timeout
server.tomcat.connection-timeout=20000

# Keep-alive
server.tomcat.keep-alive-timeout=60000
server.tomcat.max-keep-alive-requests=100
```

**Actuator Metrics:**
```bash
# Check thread pool metrics
curl http://localhost:8585/actuator/metrics/tomcat.threads.busy
curl http://localhost:8585/actuator/metrics/tomcat.threads.current
curl http://localhost:8585/actuator/metrics/tomcat.threads.config.max
```

**Load Test:**
```bash
# Test with 2000 req/s for 5 minutes
ab -n 600000 -c 2000 -t 300 "http://localhost:8585/open/volunteer/citizens?..."

# Monitor thread usage
watch -n 1 'curl -s http://localhost:8585/actuator/metrics/tomcat.threads.busy | jq'
```

**Definition of Done:**
- Tomcat settings tuned
- Load test passes (2000 req/s, no 503 errors)
- Thread pool metrics available
- Grafana dashboard updated
- Code review approved

---

### Story 3.3: Implement Pagination for Citizen Download

**Story ID:** SCALE-303  
**Story Points:** 8  
**Priority:** Critical  
**Assignee:** Backend Developer + Mobile Developer

**User Story:**
As a backend developer, I need to implement pagination for the citizen download endpoint so that response sizes are reduced from 10MB to 1MB, reducing memory pressure and enabling progressive loading.

**Acceptance Criteria:**
- [ ] Add pagination parameters (page, size) to `/volunteer/citizens` endpoint
- [ ] Enforce maximum page size of 1000 records
- [ ] Return pagination metadata (page, size, hasMore)
- [ ] Update mobile app to fetch data in pages
- [ ] Test pagination with 10K+ records
- [ ] Verify memory usage reduction
- [ ] Load test with pagination (2000 req/s)

**Technical Details:**

**VolunteerController.java:**
```java
@Cacheable(
    value = "citizens",
    key = "#state + '_' + #assemblyNo + '_' + #wardNo + '_' + #boothNos + '_' + #date + '_' + #page + '_' + #size"
)
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public Map<String, Object> citizens(
    @RequestParam String state,
    @RequestParam String assemblyNo,
    @RequestParam String wardNo,
    @RequestParam String date,
    @RequestParam String boothNos,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "1000") int size) {
    
    // Enforce max page size
    if (size > 1000) {
        size = 1000;
    }
    
    log.info("Fetching citizens: state={}, assemblyNo={}, wardNo={}, page={}, size={}", 
        state, assemblyNo, wardNo, page, size);
    
    String sql = buildCitizenQuery(state, assemblyNo, wardNo, date, boothNos);
    sql += " LIMIT " + size + " OFFSET " + (page * size);
    
    List<Map<String, Object>> citizens = replicaJdbcTemplate.query(sql, new CitizenRowMapper());
    
    // Build response
    Map<String, Object> response = new HashMap<>();
    response.put("citizens", citizens);
    response.put("page", page);
    response.put("size", size);
    response.put("hasMore", citizens.size() == size);
    response.put("count", citizens.size());
    
    return response;
}
```

**Mobile App (IONICAPP/src/providers/common.service.ts):**
```typescript
getCitizens(state, assemblyNo, wardNo, date, boothNo, data, page = 0, size = 1000) {
  var headers = new Headers();
  headers.append("Content-Type", "application/json");
  let params = new URLSearchParams();
  params.append("state", state);
  params.append("assemblyNo", assemblyNo);
  params.append("wardNo", wardNo);
  params.append("date", date);
  params.append("boothNos", boothNo);
  params.append("page", page.toString());
  params.append("size", size.toString());

  let options_n = new RequestOptions({ headers: headers, params: params });
  return this.http
    .post(this.baseUrl + "/citizens", data, options_n)
    .map((res: Response) => res.json());
}
```

**Mobile App (IONICAPP/src/providers/localdatasync.service.ts):**
```typescript
syncCitizen() {
  return new Promise((resolve, reject) => {
    this.loading = this.loadingCtrl.create({
      content: "Syncing citizens...",
    });
    this.loading.present();
    
    let page = 0;
    let allCitizens = [];
    
    const fetchPage = () => {
      this.myCitizenDatabase.getDataToSync().then((localUpdates) => {
        this.commonService.getCitizens(
          this.stateName, 
          this.assemblyNo, 
          this.wardNo, 
          this.last_synch_date, 
          -1, 
          localUpdates, 
          page, 
          1000
        ).subscribe(res => {
          allCitizens = allCitizens.concat(res.citizens);
          
          // Update progress
          this.loading.setContent(`Syncing citizens... (${allCitizens.length} loaded)`);
          
          if (res.hasMore) {
            page++;
            fetchPage();
          } else {
            // All pages fetched, save to local DB
            this.myCitizenDatabase.addUser(allCitizens).then(() => {
              this.loading.dismissAll();
              this.presentToast("Citizen Synced");
              this.syncSatusCitizen = true;
              this.checkStatus();
              resolve(true);
            }).catch(e => {
              this.loading.dismissAll();
              this.presentToast("Citizen Syncing failed");
              reject(e);
            });
          }
        }, err => {
          this.loading.dismissAll();
          this.presentToast("Citizen Syncing failed");
          reject(err);
        });
      });
    };
    
    fetchPage();
  });
}
```

**Testing:**
```bash
# Test pagination
curl "http://localhost:8585/open/volunteer/citizens?state=Karnataka&assemblyNo=123&wardNo=5&date=-1&boothNos=-1&page=0&size=1000"
curl "http://localhost:8585/open/volunteer/citizens?state=Karnataka&assemblyNo=123&wardNo=5&date=-1&boothNos=-1&page=1&size=1000"

# Load test
ab -n 20000 -c 2000 "http://localhost:8585/open/volunteer/citizens?state=Karnataka&assemblyNo=123&wardNo=5&date=-1&boothNos=-1&page=0&size=1000"
```

**Definition of Done:**
- Pagination implemented in backend
- Mobile app updated to use pagination
- Response size reduced from 10MB to 1MB
- Memory usage reduced (verified via JVM metrics)
- Load test passes (2000 req/s, p95 < 500ms)
- Code review approved
- Merged to main branch

---

### Story 3.4: Optimize Queries with DTO Projections

**Story ID:** SCALE-304  
**Story Points:** 5  
**Priority:** High  
**Assignee:** Backend Developer

**User Story:**
As a backend developer, I need to use DTO projections instead of SELECT * so that response payloads are reduced by 40%, decreasing bandwidth and serialization time.

**Acceptance Criteria:**
- [ ] Create CitizenListDTO with only essential fields
- [ ] Update citizen query to select only required columns
- [ ] Remove heavy fields (address, latitude, longitude) from list endpoint
- [ ] Measure response size reduction (target 40%)
- [ ] Verify no functional regression
- [ ] Load test with optimized queries

**Technical Details:**

**CitizenListDTO.java (create):**
```java
public class CitizenListDTO {
    private Long id;
    private String voterId;
    private String firstName;
    private String familyName;
    private String gender;
    private Integer age;
    private String mobile;
    private String srNo;
    private String boothNo;
    private String wardNo;
    private String respondedStatus;
    private Boolean voted;
    private Date modifiedDate;
    
    // Getters and setters
}
```

**CitizenRowMapper.java (update):**
```java
public class CitizenListRowMapper implements RowMapper<CitizenListDTO> {
    @Override
    public CitizenListDTO mapRow(ResultSet rs, int rowNum) throws SQLException {
        CitizenListDTO dto = new CitizenListDTO();
        dto.setId(rs.getLong("id"));
        dto.setVoterId(rs.getString("voter_id"));
        dto.setFirstName(rs.getString("first_name"));
        dto.setFamilyName(rs.getString("family_name"));
        dto.setGender(rs.getString("gender"));
        dto.setAge(rs.getInt("age"));
        dto.setMobile(rs.getString("mobile"));
        dto.setSrNo(rs.getString("srno"));
        dto.setBoothNo(rs.getString("booth_no"));
        dto.setWardNo(rs.getString("ward_no"));
        dto.setRespondedStatus(rs.getString("responded_status"));
        dto.setVoted(rs.getBoolean("voted"));
        dto.setModifiedDate(rs.getTimestamp("modifieddate"));
        return dto;
    }
}
```

**VolunteerController.java (update):**
```java
private String buildCitizenQuery(String state, String assemblyNo, String wardNo, String date, String boothNos) {
    // Use projection instead of SELECT *
    StringBuilder sql = new StringBuilder();
    sql.append("SELECT ");
    sql.append("C.id, C.voter_id, C.first_name, C.family_name, C.gender, C.age, ");
    sql.append("C.mobile, C.srno, C.booth_no, C.ward_no, C.responded_status, C.voted, ");
    sql.append("C.modifieddate ");
    sql.append("FROM citizen C ");
    sql.append("WHERE C.state = '").append(state).append("' ");
    sql.append("AND C.assembly_no = '").append(assemblyNo).append("' ");
    
    if (!"-1".equals(wardNo)) {
        sql.append("AND C.ward_no = '").append(wardNo).append("' ");
    }
    if (!"-1".equals(boothNos)) {
        sql.append("AND C.booth_no IN (").append(boothNos).append(") ");
    }
    if (!"-1".equals(date)) {
        Date dateObj = new Date(Long.parseLong(date));
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        sql.append("AND C.modifieddate > '").append(sdf.format(dateObj)).append("' ");
    }
    
    sql.append("ORDER BY C.srno + 0");
    
    return sql.toString();
}

@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public Map<String, Object> citizens(...) {
    String sql = buildCitizenQuery(state, assemblyNo, wardNo, date, boothNos);
    sql += " LIMIT " + size + " OFFSET " + (page * size);
    
    List<CitizenListDTO> citizens = replicaJdbcTemplate.query(sql, new CitizenListRowMapper());
    
    // Build response
    Map<String, Object> response = new HashMap<>();
    response.put("citizens", citizens);
    response.put("page", page);
    response.put("size", size);
    response.put("hasMore", citizens.size() == size);
    
    return response;
}
```

**Testing:**
```bash
# Compare response sizes
# Before (SELECT *)
curl "http://localhost:8585/open/volunteer/citizens?..." | wc -c

# After (DTO projection)
curl "http://localhost:8585/open/volunteer/citizens?..." | wc -c

# Should see 40% reduction
```

**Definition of Done:**
- DTO projection implemented
- Response size reduced by 40%
- No functional regression
- Load test passes
- Code review approved

---

### Story 3.5: Enforce Mandatory Filters

**Story ID:** SCALE-305  
**Story Points:** 2  
**Priority:** High  
**Assignee:** Backend Developer

**User Story:**
As a backend developer, I need to enforce mandatory filters (assemblyNo or wardNo) on citizen queries so that broad scans on 10M+ rows are prevented.

**Acceptance Criteria:**
- [ ] Add validation to reject requests without assemblyNo or wardNo
- [ ] Return 400 Bad Request with clear error message
- [ ] Update API documentation
- [ ] Test validation logic
- [ ] Verify mobile app always sends required filters

**Technical Details:**

**VolunteerController.java:**
```java
@RequestMapping(value = "/volunteer/citizens", method = RequestMethod.GET)
@ResponseBody
public Map<String, Object> citizens(
    @RequestParam String state,
    @RequestParam String assemblyNo,
    @RequestParam(required = false, defaultValue = "-1") String wardNo,
    @RequestParam String date,
    @RequestParam String boothNos,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "1000") int size) {
    
    // Enforce mandatory filters
    if (Strings.isBlank(assemblyNo) || "-1".equals(assemblyNo)) {
        if (Strings.isBlank(wardNo) || "-1".equals(wardNo)) {
            throw new ResponseStatusException(
                HttpStatus.BAD_REQUEST, 
                "assemblyNo or wardNo is required. Broad queries are not allowed."
            );
        }
    }
    
    // Existing logic
    ...
}
```

**Testing:**
```bash
# Test validation
curl "http://localhost:8585/open/volunteer/citizens?state=Karnataka&assemblyNo=-1&wardNo=-1&date=-1&boothNos=-1"
# Should return 400 Bad Request

curl "http://localhost:8585/open/volunteer/citizens?state=Karnataka&assemblyNo=123&wardNo=-1&date=-1&boothNos=-1"
# Should return 200 OK
```

**Definition of Done:**
- Validation implemented
- 400 error returned for invalid requests
- Mobile app verified to send required filters
- API documentation updated
- Code review approved

---

### Story 3.6: Stream CSV File Downloads

**Story ID:** SCALE-306  
**Story Points:** 5  
**Priority:** Medium  
**Assignee:** Backend Developer

**User Story:**
As a backend developer, I need to stream CSV downloads directly to the response so that file I/O doesn't block request threads and cause memory pressure.

**Acceptance Criteria:**
- [ ] Refactor `/getVotersDataCSV` to stream directly to response
- [ ] Remove file buffering to local filesystem
- [ ] Test CSV download with 100K+ records
- [ ] Verify memory usage doesn't spike during download
- [ ] Support concurrent downloads (10+ simultaneous)

**Technical Details:**

**MobileController.java:**
```java
@RequestMapping(value = "/getVotersDataCSV", method = RequestMethod.GET)
public void getVotersDataCSV(
    HttpServletRequest request, 
    HttpServletResponse response) throws IOException {
    
    log.info("In getVotersDataCSV()");
    String fileName = request.getParameter("fileName");
    String id = request.getParameter("id");
    fileName = Objects.nonNull(fileName) && !fileName.isEmpty() ? fileName : "Voters.csv";
    
    response.setContentType("text/csv");
    response.addHeader("content-disposition", "attachment; filename=\"" + fileName + "\"");
    
    try (OutputStream out = response.getOutputStream();
         OutputStreamWriter writer = new OutputStreamWriter(out, StandardCharsets.UTF_8)) {
        
        // Write CSV header
        writer.write("Assembly Constituency,Ward,Booth,SL No,Voter ID,First Name,Family Name,Age,Gender,Mobile,Status,Voted\n");
        
        // Stream query results directly to CSV
        String sql = "SELECT assembly_no, ward_no, booth_no, srno, voter_id, first_name, " +
                    "family_name, age, gender, mobile, responded_status, voted " +
                    "FROM citizen WHERE assembly_no = ? ORDER BY srno + 0";
        
        replicaJdbcTemplate.query(sql, new Object[]{id}, rs -> {
            // Write each row directly to output stream
            writer.write(String.format("%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s\n",
                escapeCSV(rs.getString("assembly_no")),
                escapeCSV(rs.getString("ward_no")),
                escapeCSV(rs.getString("booth_no")),
                escapeCSV(rs.getString("srno")),
                escapeCSV(rs.getString("voter_id")),
                escapeCSV(rs.getString("first_name")),
                escapeCSV(rs.getString("family_name")),
                rs.getInt("age"),
                escapeCSV(rs.getString("gender")),
                escapeCSV(rs.getString("mobile")),
                escapeCSV(rs.getString("responded_status")),
                rs.getBoolean("voted")
            ));
            writer.flush(); // Flush periodically
        });
        
    } catch (IOException e) {
        log.error("Error streaming CSV: " + e.getMessage(), e);
        throw e;
    }
}

private String escapeCSV(String value) {
    if (value == null) return "";
    if (value.contains(",") || value.contains("\"") || value.contains("\n")) {
        return "\"" + value.replace("\"", "\"\"") + "\"";
    }
    return value;
}
```

**Testing:**
```bash
# Test CSV download
curl "http://localhost:8585/open/getVotersDataCSV?id=123&fileName=voters.csv" -o voters.csv

# Test concurrent downloads
for i in {1..10}; do
  curl "http://localhost:8585/open/getVotersDataCSV?id=123&fileName=voters_$i.csv" -o voters_$i.csv &
done
wait

# Monitor memory usage during download
jstat -gc <pid> 1000
```

**Definition of Done:**
- CSV streaming implemented
- No file buffering to disk
- Memory usage stable during download
- Concurrent downloads supported (10+)
- Code review approved

---

## Phase 4: Containerization & Kubernetes (Week 7-8)

### Story 4.1: Dockerize Backend Application

**Story ID:** SCALE-401  
**Story Points:** 5  
**Priority:** Critical  
**Assignee:** DevOps Engineer + Backend Developer

**User Story:**
As a DevOps engineer, I need to containerize the Spring Boot backend so that it can be deployed on Kubernetes with consistent environments and easy scaling.

**Acceptance Criteria:**
- [ ] Create Dockerfile with JVM tuning for containers
- [ ] Build Docker image
- [ ] Test Docker image locally
- [ ] Push image to container registry
- [ ] Document build and run commands
- [ ] Verify application health checks work in container

**Technical Details:**

**Dockerfile:**
```dockerfile
FROM openjdk:8-jre-alpine

# Install curl for health checks
RUN apk add --no-cache curl

WORKDIR /app

# Copy JAR
COPY target/smartielection-backend.jar app.jar

# Expose port
EXPOSE 8585

# JVM tuning for containers
ENV JAVA_OPTS="-Xms2g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+UseStringDeduplication \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/logs/heap_dump.hprof \
  -Djava.security.egd=file:/dev/./urandom"

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8585/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**.dockerignore:**
```
target/
.git/
.idea/
*.log
*.md
Dockerfile
docker-compose.yml
```

**Build Script (build-docker.sh):**
```bash
#!/bin/bash
set -e

echo "Building application..."
mvn clean package -DskipTests

echo "Building Docker image..."
docker build -t smartneta-backend:latest .

echo "Tagging image..."
docker tag smartneta-backend:latest <registry>/smartneta-backend:latest

echo "Pushing to registry..."
docker push <registry>/smartneta-backend:latest

echo "Done!"
```

**Test Locally:**
```bash
# Build
./build-docker.sh

# Run
docker run -d \
  --name smartneta-backend \
  -p 8585:8585 \
  -e SPRING_DATASOURCE_PRIMARY_URL="jdbc:mysql://host.docker.internal:3306/sampark?..." \
  -e SPRING_REDIS_HOST="host.docker.internal" \
  smartneta-backend:latest

# Check logs
docker logs -f smartneta-backend

# Test health check
curl http://localhost:8585/actuator/health

# Stop
docker stop smartneta-backend
docker rm smartneta-backend
```

**Definition of Done:**
- Dockerfile created and tested
- Docker image builds successfully
- Application runs in container
- Health checks work
- Image pushed to registry
- Documentation complete
- Code review approved

---

### Story 4.2: Create Kubernetes Deployment Manifests

**Story ID:** SCALE-402  
**Story Points:** 8  
**Priority:** Critical  
**Assignee:** DevOps Engineer  
**Dependencies:** SCALE-401

**User Story:**
As a DevOps engineer, I need to create Kubernetes deployment manifests so that the backend can be deployed with 5-20 replicas, auto-scaling, and zero-downtime updates.

**Acceptance Criteria:**
- [ ] Create Deployment manifest with 5 initial replicas
- [ ] Create Service manifest (LoadBalancer)
- [ ] Create HorizontalPodAutoscaler (5-20 replicas)
- [ ] Create ConfigMap for application properties
- [ ] Create Secret for database passwords
- [ ] Configure resource requests/limits
- [ ] Configure liveness and readiness probes
- [ ] Test deployment on staging cluster

**Technical Details:**

**k8s/namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: smartneta
```

**k8s/configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: smartneta-config
  namespace: smartneta
data:
  application.properties: |
    spring.datasource.primary.url=jdbc:mysql://mysql-primary:3306/sampark?useSSL=false&serverTimezone=UTC
    spring.datasource.replica.url=jdbc:mysql://mysql-replica-1:3306/sampark?useSSL=false&serverTimezone=UTC
    spring.datasource.reporting.url=jdbc:mysql://mysql-reporting:3306/sampark?useSSL=false&serverTimezone=UTC
    spring.redis.host=redis
    spring.redis.port=6379
    server.port=8585
```

**k8s/secret.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: smartneta-secrets
  namespace: smartneta
type: Opaque
stringData:
  db-primary-password: "mypass"
  db-replica-password: "readonly"
  db-reporting-password: "readonly"
```

**k8s/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smartneta-backend
  namespace: smartneta
  labels:
    app: smartneta-backend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: smartneta-backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: smartneta-backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8585"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
      - name: backend
        image: <registry>/smartneta-backend:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8585
          name: http
        env:
        - name: SPRING_DATASOURCE_PRIMARY_PASSWORD
          valueFrom:
            secretKeyRef:
              name: smartneta-secrets
              key: db-primary-password
        - name: SPRING_DATASOURCE_REPLICA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: smartneta-secrets
              key: db-replica-password
        - name: SPRING_DATASOURCE_REPORTING_PASSWORD
          valueFrom:
            secretKeyRef:
              name: smartneta-secrets
              key: db-reporting-password
        - name: JAVA_OPTS
          value: "-Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8585
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8585
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: smartneta-config
```

**k8s/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: smartneta-backend
  namespace: smartneta
spec:
  type: LoadBalancer
  selector:
    app: smartneta-backend
  ports:
  - port: 80
    targetPort: 8585
    protocol: TCP
    name: http
  sessionAffinity: None
```

**k8s/hpa.yaml:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: smartneta-backend-hpa
  namespace: smartneta
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: smartneta-backend
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

**Deploy Script (deploy.sh):**
```bash
#!/bin/bash
set -e

echo "Creating namespace..."
kubectl apply -f k8s/namespace.yaml

echo "Creating secrets..."
kubectl apply -f k8s/secret.yaml

echo "Creating configmap..."
kubectl apply -f k8s/configmap.yaml

echo "Deploying application..."
kubectl apply -f k8s/deployment.yaml

echo "Creating service..."
kubectl apply -f k8s/service.yaml

echo "Creating HPA..."
kubectl apply -f k8s/hpa.yaml

echo "Waiting for rollout..."
kubectl rollout status deployment/smartneta-backend -n smartneta

echo "Getting service URL..."
kubectl get svc smartneta-backend -n smartneta

echo "Done!"
```

**Testing:**
```bash
# Deploy
./deploy.sh

# Check pods
kubectl get pods -n smartneta

# Check HPA
kubectl get hpa -n smartneta

# Check logs
kubectl logs -f deployment/smartneta-backend -n smartneta

# Test endpoint
LOAD_BALANCER_IP=$(kubectl get svc smartneta-backend -n smartneta -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$LOAD_BALANCER_IP/actuator/health
```

**Definition of Done:**
- All K8s manifests created
- Deployment successful on staging
- 5 pods running
- HPA configured and working
- Service accessible via LoadBalancer
- Logs and metrics available
- Documentation complete
- Code review approved

---

### Story 4.3: Deploy MySQL and Redis on Kubernetes

**Story ID:** SCALE-403  
**Story Points:** 8  
**Priority:** Critical  
**Assignee:** DevOps Engineer  
**Dependencies:** SCALE-402

**User Story:**
As a DevOps engineer, I need to deploy MySQL (primary + replicas) and Redis on Kubernetes so that the entire stack runs on K8s with persistent storage.

**Acceptance Criteria:**
- [ ] Create StatefulSet for MySQL primary
- [ ] Create StatefulSets for MySQL replicas (2)
- [ ] Configure MySQL replication
- [ ] Create PersistentVolumeClaims for MySQL (500GB each)
- [ ] Deploy Redis with persistent storage
- [ ] Test database connectivity from backend pods
- [ ] Verify replication working
- [ ] Document backup/restore procedures

**Technical Details:**

**k8s/mysql-primary-statefulset.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-primary
  namespace: smartneta
spec:
  ports:
  - port: 3306
  clusterIP: None
  selector:
    app: mysql-primary
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-primary
  namespace: smartneta
spec:
  serviceName: mysql-primary
  replicas: 1
  selector:
    matchLabels:
      app: mysql-primary
  template:
    metadata:
      labels:
        app: mysql-primary
    spec:
      containers:
      - name: mysql
        image: mysql:8.0.32
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: smartneta-secrets
              key: db-primary-password
        - name: MYSQL_DATABASE
          value: "sampark"
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            memory: "8Gi"
            cpu: "2000m"
          limits:
            memory: "16Gi"
            cpu: "4000m"
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "localhost", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 500Gi
```

**k8s/redis-deployment.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: smartneta
spec:
  ports:
  - port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: smartneta
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
          - redis-server
          - --maxmemory
          - "4gb"
          - --maxmemory-policy
          - allkeys-lru
          - --save
          - "900"
          - "1"
          - --appendonly
          - "yes"
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "1000m"
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: smartneta
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

**Deploy:**
```bash
# Deploy MySQL
kubectl apply -f k8s/mysql-primary-statefulset.yaml

# Deploy Redis
kubectl apply -f k8s/redis-deployment.yaml

# Check status
kubectl get pods -n smartneta
kubectl get pvc -n smartneta

# Test MySQL connection
kubectl exec -it mysql-primary-0 -n smartneta -- mysql -u root -p

# Test Redis connection
kubectl exec -it <redis-pod> -n smartneta -- redis-cli PING
```

**Definition of Done:**
- MySQL and Redis deployed on K8s
- Persistent storage configured
- Database connectivity tested
- Replication verified (if replicas deployed)
- Backup procedures documented
- Code review approved

---

### Story 4.4: Load Testing and Performance Tuning

**Story ID:** SCALE-404  
**Story Points:** 13  
**Priority:** Critical  
**Assignee:** Backend Developer + DevOps Engineer  
**Dependencies:** SCALE-402, SCALE-403

**User Story:**
As a DevOps engineer, I need to perform progressive load testing and tune the system so that it can reliably handle 2750 req/s with p95 < 500ms and < 0.5% error rate.

**Acceptance Criteria:**
- [ ] Set up load testing infrastructure (JMeter/k6)
- [ ] Run baseline test (500 req/s)
- [ ] Run ramp-up test (1000 req/s)
- [ ] Run target test (2750 req/s)
- [ ] Run spike test (5000 req/s)
- [ ] Run endurance test (2750 req/s for 1 hour)
- [ ] Tune system based on bottlenecks identified
- [ ] Document performance metrics
- [ ] Create Grafana dashboards for monitoring

**Technical Details:**

**Load Test Script (k6/load-test.js):**
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export let options = {
  stages: [
    { duration: '2m', target: 500 },   // Ramp up to 500 RPS
    { duration: '5m', target: 500 },   // Stay at 500 RPS
    { duration: '2m', target: 1000 },  // Ramp up to 1000 RPS
    { duration: '5m', target: 1000 },  // Stay at 1000 RPS
    { duration: '2m', target: 2750 },  // Ramp up to 2750 RPS
    { duration: '10m', target: 2750 }, // Stay at 2750 RPS
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500'], // 95% of requests must complete below 500ms
    'errors': ['rate<0.005'],           // Error rate must be below 0.5%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8585';

export default function () {
  const params = {
    headers: {
      'Content-Type': 'application/json',
    },
  };

  // Citizen download request
  const res = http.get(
    `${BASE_URL}/open/volunteer/citizens?state=Karnataka&assemblyNo=123&wardNo=5&date=-1&boothNos=-1&page=0&size=1000`,
    params
  );

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  }) || errorRate.add(1);

  sleep(0.1); // Small delay between requests
}
```

**Run Load Tests:**
```bash
# Install k6
brew install k6  # macOS
# or
sudo apt-get install k6  # Ubuntu

# Run baseline test (500 RPS)
k6 run --vus 500 --duration 10m k6/load-test-baseline.js

# Run target test (2750 RPS)
k6 run --vus 2750 --duration 10m k6/load-test-target.js

# Run endurance test (2750 RPS for 1 hour)
k6 run --vus 2750 --duration 1h k6/load-test-endurance.js

# Run spike test (5000 RPS for 2 minutes)
k6 run --vus 5000 --duration 2m k6/load-test-spike.js
```

**Monitoring During Load Test:**
```bash
# Watch HPA scaling
watch -n 5 'kubectl get hpa -n smartneta'

# Watch pod CPU/memory
watch -n 5 'kubectl top pods -n smartneta'

# Watch pod count
watch -n 5 'kubectl get pods -n smartneta | grep smartneta-backend | wc -l'

# Check Redis hit rate
kubectl exec -it <redis-pod> -n smartneta -- redis-cli INFO stats | grep keyspace

# Check MySQL connections
kubectl exec -it mysql-primary-0 -n smartneta -- mysql -u root -p -e "SHOW PROCESSLIST;"
```

**Grafana Dashboards:**
- Request rate (req/s)
- Response time (p50, p95, p99)
- Error rate (%)
- Pod count and CPU/memory usage
- Database connection pool usage
- Redis hit rate
- MySQL query latency

**Tuning Iterations:**

1. **If HPA doesn't scale fast enough:**
   - Adjust `scaleUp.periodSeconds` to 15s
   - Increase `scaleUp.policies.value` to 200%

2. **If database connection pool saturated:**
   - Increase HikariCP `maximum-pool-size`
   - Add more MySQL replicas

3. **If Redis memory exhausted:**
   - Increase Redis maxmemory to 8GB
   - Adjust cache TTLs

4. **If pods OOM killed:**
   - Increase memory limits to 6Gi
   - Tune JVM -Xmx to 5g

5. **If response time > 500ms:**
   - Check slow query log
   - Verify cache hit rate > 70%
   - Check for N+1 queries

**Definition of Done:**
- All load tests passed
- 2750 req/s sustained for 1 hour
- p95 response time < 500ms
- Error rate < 0.5%
- HPA scaling working correctly
- Grafana dashboards created
- Performance report documented
- System tuned and optimized
- Code review approved

---

### Story 4.5: Set Up Production Monitoring and Alerting

**Story ID:** SCALE-405  
**Story Points:** 5  
**Priority:** High  
**Assignee:** DevOps Engineer  
**Dependencies:** SCALE-404

**User Story:**
As a DevOps engineer, I need to set up comprehensive monitoring and alerting so that performance issues are detected and alerted before they impact users.

**Acceptance Criteria:**
- [ ] Deploy Prometheus for metrics collection
- [ ] Deploy Grafana for visualization
- [ ] Create dashboards for key metrics
- [ ] Configure alerts for critical thresholds
- [ ] Set up log aggregation (ELK/Loki)
- [ ] Test alert notifications (Slack/email)
- [ ] Document monitoring setup

**Technical Details:**

**Prometheus Configuration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    
    scrape_configs:
    - job_name: 'smartneta-backend'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - smartneta
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
```

**Grafana Dashboards:**
1. **Application Performance Dashboard:**
   - Request rate (req/s)
   - Response time (p50, p95, p99)
   - Error rate (%)
   - Active requests

2. **Infrastructure Dashboard:**
   - Pod count
   - CPU usage per pod
   - Memory usage per pod
   - Network I/O

3. **Database Dashboard:**
   - Connection pool usage (active, idle, pending)
   - Query latency (p50, p95, p99)
   - Slow query count
   - Replication lag

4. **Cache Dashboard:**
   - Redis hit rate
   - Redis memory usage
   - Cache evictions
   - Keys count

**Alerts:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: monitoring
data:
  alerts.yml: |
    groups:
    - name: smartneta
      interval: 30s
      rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors/sec"
      
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "P95 response time is {{ $value }}s"
      
      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_pending > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool exhausted"
          description: "{{ $value }} connections pending"
      
      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage high"
          description: "Redis memory usage is {{ $value }}%"
      
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod is crash looping"
          description: "Pod {{ $labels.pod }} has restarted {{ $value }} times"
```

**Definition of Done:**
- Prometheus and Grafana deployed
- Dashboards created and accessible
- Alerts configured and tested
- Log aggregation set up
- Runbook documented
- Team trained on monitoring tools
- Code review approved

---

## Summary

**Total Stories:** 25  
**Total Story Points:** 130  
**Duration:** 8 weeks  
**Team:** 2 Backend Developers + 1 DevOps Engineer

### Phase Breakdown:
- **Phase 1 (Database Layer):** 4 stories, 21 points, 2 weeks
- **Phase 2 (Caching Layer):** 5 stories, 19 points, 2 weeks
- **Phase 3 (Backend Tuning):** 6 stories, 25 points, 2 weeks
- **Phase 4 (K8s Deployment):** 5 stories, 39 points, 2 weeks
- **Monitoring:** 1 story, 5 points, ongoing

### Success Criteria:
- ✅ Sustained 2750 req/s for 1 hour
- ✅ p95 response time < 500ms
- ✅ Error rate < 0.5%
- ✅ Auto-scaling from 5 to 20 pods
- ✅ Zero downtime deployments
- ✅ Cache hit rate > 70%
- ✅ Infrastructure cost < $2,500/month

### Risk Mitigation:
- Each story is independently testable
- Progressive load testing ensures early detection of issues
- Rollback procedures documented for each phase
- Monitoring and alerting in place before production launch

