# Individual-Level AggregateIndividual Processing Service Proposal

## Executive Summary

This proposal outlines how to create a new **Individual Processing Service** that can create one or a few `AggregateIndividual` objects on-demand using direct database lookups. This service will be consumed by other services that need real-time individual data processing without requiring full tree batch processing.

## Current System Analysis

### Current Batch Processing Flow

The existing system follows this pipeline:
```
MySQL Database → Sqoop Export → Avro Files → TreeAggregator → AggregatedTree → AggregateIndividualFromTreeProcessor → AggregateIndividual[]
```

**Key Components:**
1. **TreeAggregator**: Reads Avro files and creates `AggregatedTree` objects containing entire family trees
2. **AggregateIndividualFromTreeProcessor**: Processes complete trees to create all `AggregateIndividual` objects
3. **RelativesBuilder**: Builds relationships using complete tree context (all individuals, families, events)

**Current Limitations for Individual Processing:**
- Requires complete tree context for relationship building
- Processes entire trees at once (tree-centric approach)
- No direct database connectivity for on-demand lookups
- Memory-intensive for large trees

## Proposed Individual Processing Service

### Service Architecture Overview

```
Client Services → REST API → Individual Processing Service → Database Accessor → MySQL Database
                     ↓
              AggregateIndividual Response
```

### Service Design Principles

1. **On-Demand Processing**: Process individuals in real-time based on service requests
2. **RESTful API**: Expose clean HTTP endpoints for other services to consume
3. **Selective Data Fetching**: Only fetch data necessary for the requested individual(s) and their immediate relationships
4. **Database-Direct**: Bypass Avro files and query source database tables directly
5. **Relationship-Aware**: Intelligently fetch related individuals needed for relationship building
6. **High Performance**: Sub-second response times with proper caching
7. **Scalable**: Handle concurrent requests from multiple client services
8. **Format Compatible**: Return standard `AggregateIndividual` and `SearchableIndividual` objects

## Implementation Design

### 1. REST API Design

#### Service Endpoints

**Base URL**: `http://individual-processor-service:8080/api/v1`

##### Core Individual Processing Endpoints

```http
# Process single individual
GET /individuals/{siteId}/{individualId}/aggregate
GET /individuals/{siteId}/{individualId}/searchable

# Process multiple individuals (batch)
POST /individuals/aggregate
POST /individuals/searchable

# Health and status
GET /health
GET /metrics
```

##### API Specifications

**Single Individual Processing**
```http
GET /individuals/{siteId}/{individualId}/aggregate

Response 200:
{
  "individualId": 123456,
  "siteId": 1,
  "treeId": 789,
  "firstName": "John",
  "lastName": "Smith",
  "relatives": [
    {
      "relativeIndividualId": 123457,
      "relationship": "spouse",
      "firstName": "Jane",
      "lastName": "Smith"
    }
  ],
  "events": [...],
  "processingTimeMs": 245
}

Error 404:
{
  "error": "INDIVIDUAL_NOT_FOUND",
  "message": "Individual 123456 not found in site 1",
  "timestamp": "2025-09-11T10:30:00Z"
}

Error 500:
{
  "error": "PROCESSING_ERROR", 
  "message": "Database connection failed",
  "timestamp": "2025-09-11T10:30:00Z"
}
```

**Batch Individual Processing**
```http
POST /individuals/aggregate

Request Body:
{
  "individuals": [
    {"siteId": 1, "individualId": 123456},
    {"siteId": 1, "individualId": 123457},
    {"siteId": 2, "individualId": 789012}
  ],
  "options": {
    "includeExtendedRelationships": true,
    "maxRelationshipDepth": 2
  }
}

Response 200:
{
  "results": [
    {
      "siteId": 1,
      "individualId": 123456, 
      "status": "SUCCESS",
      "data": { /* AggregateIndividual object */ }
    },
    {
      "siteId": 1,
      "individualId": 123457,
      "status": "SUCCESS", 
      "data": { /* AggregateIndividual object */ }
    },
    {
      "siteId": 2,
      "individualId": 789012,
      "status": "ERROR",
      "error": "INDIVIDUAL_NOT_FOUND"
    }
  ],
  "summary": {
    "totalRequested": 3,
    "successful": 2,
    "failed": 1,
    "processingTimeMs": 432
  }
}
```

#### Service Implementation Framework

**Using Dropwizard/Jersey for Java REST Service**
```java
@Path("/api/v1/individuals")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class IndividualProcessingResource {
    
    private final IndividualProcessingService processingService;
    private final MetricRegistry metrics;
    
    @GET
    @Path("/{siteId}/{individualId}/aggregate")
    @Timed(name = "individual-aggregate-processing")
    public Response getAggregateIndividual(
            @PathParam("siteId") long siteId,
            @PathParam("individualId") long individualId,
            @QueryParam("includeExtended") @DefaultValue("true") boolean includeExtended) {
        
        try {
            AggregateIndividual result = processingService.processIndividual(
                siteId, individualId, includeExtended);
            
            return Response.ok(result).build();
            
        } catch (IndividualNotFoundException e) {
            return Response.status(404)
                .entity(new ErrorResponse("INDIVIDUAL_NOT_FOUND", e.getMessage()))
                .build();
        } catch (Exception e) {
            LOGGER.error("Failed to process individual {}-{}", siteId, individualId, e);
            return Response.status(500)
                .entity(new ErrorResponse("PROCESSING_ERROR", "Internal server error"))
                .build();
        }
    }
    
    @GET
    @Path("/{siteId}/{individualId}/searchable")  
    @Timed(name = "individual-searchable-processing")
    public Response getSearchableIndividual(
            @PathParam("siteId") long siteId,
            @PathParam("individualId") long individualId) {
        
        try {
            SearchableIndividual result = processingService.processIndividualToSearchable(
                siteId, individualId);
            return Response.ok(result).build();
        } catch (Exception e) {
            return handleError(e, siteId, individualId);
        }
    }
    
    @POST
    @Path("/aggregate")
    @Timed(name = "batch-aggregate-processing")
    public Response batchProcessAggregateIndividuals(BatchProcessingRequest request) {
        try {
            BatchProcessingResponse response = processingService.batchProcessIndividuals(request);
            return Response.ok(response).build();
        } catch (Exception e) {
            return handleBatchError(e);
        }
    }
}
```

### 2. Service Architecture Components

#### IndividualProcessingService (Core Service Layer)
```java
@Singleton
public class IndividualProcessingService {
    
    private final IndividualDatabaseAccessor dbAccessor;
    private final IndividualAggregateProcessor processor;
    private final SearchableIndividualMapper searchableMapper;
    private final CacheManager cacheManager;
    private final MetricRegistry metrics;
    
    @Inject
    public IndividualProcessingService(
            IndividualDatabaseAccessor dbAccessor,
            CacheManager cacheManager,
            MetricRegistry metrics) {
        this.dbAccessor = dbAccessor;
        this.processor = new IndividualAggregateProcessor(dbAccessor);
        this.searchableMapper = new SearchableIndividualMapper();
        this.cacheManager = cacheManager;
        this.metrics = metrics;
    }
    
    @Timed(name = "process-individual")
    public AggregateIndividual processIndividual(long siteId, long individualId, boolean includeExtended) {
        String cacheKey = String.format("agg:%d:%d:%b", siteId, individualId, includeExtended);
        
        // Check cache first
        AggregateIndividual cached = cacheManager.get(cacheKey, AggregateIndividual.class);
        if (cached != null) {
            metrics.counter("cache.hits").inc();
            return cached;
        }
        
        // Process individual
        AggregateIndividual result = processor.processIndividual(siteId, individualId, includeExtended);
        
        // Cache result (with TTL)
        cacheManager.put(cacheKey, result, Duration.ofMinutes(30));
        metrics.counter("cache.misses").inc();
        
        return result;
    }
    
    public SearchableIndividual processIndividualToSearchable(long siteId, long individualId) {
        AggregateIndividual aggIndividual = processIndividual(siteId, individualId, true);
        return searchableMapper.map(aggIndividual);
    }
    
    @Timed(name = "batch-process-individuals")
    public BatchProcessingResponse batchProcessIndividuals(BatchProcessingRequest request) {
        List<IndividualRequest> requests = request.getIndividuals();
        List<BatchProcessingResult> results = new ArrayList<>();
        
        // Group by site for efficient database access
        Map<Long, List<IndividualRequest>> requestsBySite = requests.stream()
            .collect(Collectors.groupingBy(IndividualRequest::getSiteId));
        
        for (Map.Entry<Long, List<IndividualRequest>> entry : requestsBySite.entrySet()) {
            long siteId = entry.getKey();
            List<IndividualRequest> siteRequests = entry.getValue();
            
            try {
                // Batch process individuals from same site for efficiency
                List<AggregateIndividual> siteResults = processor.batchProcessIndividuals(
                    siteId, siteRequests.stream()
                        .map(IndividualRequest::getIndividualId)
                        .collect(Collectors.toSet()),
                    request.getOptions());
                
                // Convert to response format
                for (AggregateIndividual result : siteResults) {
                    results.add(BatchProcessingResult.success(
                        result.getSiteId(), result.getIndividualId(), result));
                }
                
            } catch (Exception e) {
                // Handle site-level errors
                for (IndividualRequest req : siteRequests) {
                    results.add(BatchProcessingResult.error(
                        req.getSiteId(), req.getIndividualId(), 
                        "PROCESSING_ERROR", e.getMessage()));
                }
            }
        }
        
        return new BatchProcessingResponse(results);
    }
}
```

### 3. Database Access Layer

#### IndividualDatabaseAccessor
```java
public class IndividualDatabaseAccessor {
    private final DataSource dataSource;
    private final Map<Long, SiteDbConfig> siteDbConfigs;
    
    // Core individual lookup
    public Individual getIndividual(long siteId, long individualId);
    
    // Relationship-aware fetchers
    public List<Individual> getIndividualsInTree(long siteId, long treeId, Set<Long> individualIds);
    public List<Family> getFamiliesForIndividuals(long siteId, long treeId, Set<Long> individualIds);
    public List<IndividualEvent> getEventsForIndividuals(long siteId, Set<Long> individualIds);
    public List<FamilyEvent> getEventsForFamilies(long siteId, Set<Long> familyIds);
    
    // Relationship traversal helpers
    public Set<Long> getRelatedIndividualIds(long siteId, long individualId, int relationshipDepth);
    public List<Family> getFamiliesWhereIndividualIsSpouse(long siteId, long individualId);
    public List<Family> getFamiliesWhereIndividualIsChild(long siteId, long individualId);
}
```

#### Database Query Strategy
Based on the database schema from `Database_to_SearchableIndividual_Relationship_Pipeline.md`:

**Primary Queries:**
```sql
-- Get target individual
SELECT * FROM genealogy_individual 
WHERE Site_ID = ? AND Individual_ID = ?;

-- Get families where individual is a spouse
SELECT * FROM genealogy_family 
WHERE Site_ID = ? AND Family_Tree_ID = ? 
  AND (Husband_ID = ? OR Wife_ID = ?);

-- Get families where individual is a child
SELECT * FROM genealogy_family f
WHERE Site_ID = ? AND Family_Tree_ID = ?
  AND Family_ID IN (
    SELECT Child_In_Family_ID FROM genealogy_individual 
    WHERE Site_ID = ? AND Individual_ID = ?
    UNION
    SELECT Adopted_Child_In_Family_ID FROM genealogy_individual 
    WHERE Site_ID = ? AND Individual_ID = ?
    UNION  
    SELECT Foster_Child_In_Family_ID FROM genealogy_individual
    WHERE Site_ID = ? AND Individual_ID = ?
  );

-- Get related individuals (parents, spouses, children, siblings)
SELECT DISTINCT i.* FROM genealogy_individual i
WHERE Site_ID = ? AND Family_Tree_ID = ?
  AND (
    -- Spouses
    Individual_ID IN (SELECT Husband_ID FROM genealogy_family WHERE Wife_ID = ?)
    OR Individual_ID IN (SELECT Wife_ID FROM genealogy_family WHERE Husband_ID = ?)
    -- Parents  
    OR Individual_ID IN (SELECT Husband_ID FROM genealogy_family WHERE Family_ID IN (...))
    OR Individual_ID IN (SELECT Wife_ID FROM genealogy_family WHERE Family_ID IN (...))
    -- Children
    OR Child_In_Family_ID IN (SELECT Family_ID FROM genealogy_family WHERE Husband_ID = ? OR Wife_ID = ?)
    -- Siblings (same Child_In_Family_ID)
    OR Child_In_Family_ID IN (SELECT Child_In_Family_ID FROM genealogy_individual WHERE Individual_ID = ?)
  );
```

### 2. Individual Processing Engine

#### IndividualAggregateProcessor
```java
public class IndividualAggregateProcessor {
    private final IndividualDatabaseAccessor dbAccessor;
    private final AggregateIndividualMapper individualMapper;
    private final RelativesBuilder relativesBuilder;
    
    /**
     * Process a single individual with minimal relationship context
     */
    public AggregateIndividual processIndividual(long siteId, long individualId) {
        return processIndividuals(siteId, Set.of(individualId)).get(0);
    }
    
    /**
     * Process multiple individuals efficiently with shared context
     */
    public List<AggregateIndividual> processIndividuals(long siteId, Set<Long> individualIds) {
        // 1. Fetch target individuals
        List<Individual> targetIndividuals = dbAccessor.getIndividualsInTree(siteId, treeId, individualIds);
        
        // 2. Determine related individuals needed for relationship building
        Set<Long> relatedIds = getRequiredRelatedIndividuals(siteId, individualIds);
        
        // 3. Fetch minimal relationship context
        IndividualProcessingContext context = buildMinimalContext(siteId, individualIds, relatedIds);
        
        // 4. Create AggregateIndividuals with relationships
        return createAggregateIndividuals(targetIndividuals, context);
    }
    
    private Set<Long> getRequiredRelatedIndividuals(long siteId, Set<Long> targetIds) {
        Set<Long> required = new HashSet<>();
        
        for (Long individualId : targetIds) {
            // Add immediate family members needed for relationship building
            required.addAll(dbAccessor.getRelatedIndividualIds(siteId, individualId, 1));
        }
        
        return required;
    }
}
```

#### IndividualProcessingContext
```java
public class IndividualProcessingContext {
    private final Map<Long, Individual> individualsById;
    private final Map<Long, Family> familiesById;
    private final Multimap<Long, IndividualEvent> eventsByIndividualId;
    private final Multimap<Long, FamilyEvent> eventsByFamilyId;
    private final long siteId;
    private final long treeId;
    
    // Minimal context for relationship building
    public static IndividualProcessingContext buildMinimal(
            IndividualDatabaseAccessor dbAccessor, 
            long siteId, 
            long treeId,
            Set<Long> targetIndividualIds,
            Set<Long> relatedIndividualIds) {
        
        Set<Long> allIds = Sets.union(targetIndividualIds, relatedIndividualIds);
        
        List<Individual> individuals = dbAccessor.getIndividualsInTree(siteId, treeId, allIds);
        List<Family> families = dbAccessor.getFamiliesForIndividuals(siteId, treeId, allIds);
        List<IndividualEvent> individualEvents = dbAccessor.getEventsForIndividuals(siteId, allIds);
        
        Set<Long> familyIds = families.stream().map(Family::getFamilyId).collect(toSet());
        List<FamilyEvent> familyEvents = dbAccessor.getEventsForFamilies(siteId, familyIds);
        
        return new IndividualProcessingContext(
            individuals.stream().collect(toMap(Individual::getIndividualId, identity())),
            families.stream().collect(toMap(Family::getFamilyId, identity())),
            groupByIndividualId(individualEvents),
            groupByFamilyId(familyEvents),
            siteId,
            treeId
        );
    }
}
```

### 3. Modified RelativesBuilder

#### Relationship Building Strategy
The existing `RelativesBuilder` assumes complete tree context. For individual processing, we need a **targeted relationship builder**:

```java
public class IndividualRelativesBuilder extends RelativesBuilder {
    
    /**
     * Build relationships for specific target individuals within minimal context
     */
    public void buildForTargetIndividuals(
            Collection<AggregateIndividual> allIndividuals,
            Set<Long> targetIndividualIds,
            IndividualProcessingContext context,
            Counters counters) {
        
        // Build indexes from minimal context
        buildIndexes(allIndividuals, context);
        
        // Build relationships only for target individuals
        buildSpousesForTargets(context.getFamilies(), targetIndividualIds);
        buildParentsChildrenForTargets(allIndividuals, targetIndividualIds);  
        buildSiblingsForTargets(allIndividuals, targetIndividualIds);
        
        // Build implied surnames only for targets
        buildImpliedLastnameForTargets(allIndividuals, targetIndividualIds, false);
        buildImpliedLastnameForTargets(allIndividuals, targetIndividualIds, true);
    }
    
    private void buildSpousesForTargets(Collection<Family> families, Set<Long> targetIds) {
        for (Family family : families) {
            long husbandId = family.getHusbandId();
            long wifeId = family.getWifeId();
            
            // Only build spouse relationships if one of the spouses is a target
            if (targetIds.contains(husbandId) || targetIds.contains(wifeId)) {
                addRelative(family.getFamilyId(), husbandId, wifeId, 
                           Relationship.RELATIONSHIP_SPOUSE, "", family.getStatus());
                addRelative(family.getFamilyId(), wifeId, husbandId, 
                           Relationship.RELATIONSHIP_SPOUSE, "", family.getStatus());
            }
        }
    }
    
    // Similar targeted approach for parent-child and sibling relationships...
}
```

### 4. Service Deployment and Configuration

#### Service Application (Dropwizard)
```java
public class IndividualProcessingApplication extends Application<IndividualProcessingConfiguration> {
    
    public static void main(String[] args) throws Exception {
        new IndividualProcessingApplication().run(args);
    }
    
    @Override
    public void initialize(Bootstrap<IndividualProcessingConfiguration> bootstrap) {
        bootstrap.addBundle(new AssetsBundle("/assets/", "/", "index.html"));
        bootstrap.addBundle(new MigrationsBundle<IndividualProcessingConfiguration>() {
            @Override
            public DataSourceFactory getDataSourceFactory(IndividualProcessingConfiguration configuration) {
                return configuration.getDataSourceFactory();
            }
        });
    }
    
    @Override
    public void run(IndividualProcessingConfiguration config, Environment environment) {
        // Database setup
        final DBIFactory factory = new DBIFactory();
        final DBI jdbi = factory.build(environment, config.getDataSourceFactory(), "mysql");
        
        // Cache setup  
        CacheManager cacheManager = setupCaching(config);
        
        // Database accessor
        IndividualDatabaseAccessor dbAccessor = new IndividualDatabaseAccessor(
            config.getSiteDbConfigs(), jdbi);
        
        // Core service
        IndividualProcessingService processingService = new IndividualProcessingService(
            dbAccessor, cacheManager, environment.metrics());
        
        // REST resources
        environment.jersey().register(new IndividualProcessingResource(processingService));
        
        // Health checks
        environment.healthChecks().register("database", 
            new DatabaseHealthCheck(dbAccessor));
        environment.healthChecks().register("cache",
            new CacheHealthCheck(cacheManager));
    }
    
    private CacheManager setupCaching(IndividualProcessingConfiguration config) {
        if (config.getRedisConfig().isEnabled()) {
            return new RedisCacheManager(config.getRedisConfig());
        } else {
            return new InMemoryCacheManager(config.getCacheConfig());
        }
    }
}
```

#### Service Configuration
```yaml
# individual-processing-service.yml
server:
  applicationConnectors:
    - type: http
      port: 8080
  adminConnectors:
    - type: http
      port: 8081

database:
  driverClass: com.mysql.cj.jdbc.Driver
  user: application
  password: ${DB_PASSWORD}
  url: jdbc:mysql://localhost:3306/genealogy
  properties:
    charSet: UTF-8
    hibernate.dialect: org.hibernate.dialect.MySQLDialect
  maxWaitForConnection: 1s
  validationQuery: "SELECT 1"
  minSize: 8
  maxSize: 32
  checkConnectionWhileIdle: false

siteDbConfigs:
  configPath: /etc/individual-processor/site-db-configs.json
  refreshIntervalMinutes: 5

caching:
  # In-memory cache configuration
  inMemory:
    maxSize: 10000
    expireAfterWrite: 30m
    expireAfterAccess: 10m
  
  # Redis cache configuration (optional)
  redis:
    enabled: false
    host: localhost
    port: 6379
    database: 0
    timeout: 2s

logging:
  level: INFO
  loggers:
    com.myheritage.individual.processor: DEBUG
  appenders:
    - type: console
    - type: file
      currentLogFilename: /var/log/individual-processor/application.log
      archivedLogFilenamePattern: /var/log/individual-processor/application-%d.log.gz
      archivedFileCount: 7

metrics:
  reporters:
    - type: graphite
      host: metrics.myheritage.com
      port: 2003
      prefix: individual-processor
```

### 5. Configuration and Setup

#### Database Configuration
```java
public class SiteDbConfigManager {
    private final Map<Long, SiteDbConfig> siteConfigs;
    
    public static SiteDbConfigManager fromFile(String configPath) {
        // Read site database configuration similar to existing SiteDatabaseConfigManager.py
        // but in Java for individual processing mode
    }
    
    public SiteDbConfig getConfigForSite(long siteId) {
        return siteConfigs.get(siteId);
    }
}

public class SiteDbConfig {
    private final String host;
    private final int port;
    private final String database;
    private final String username;
    private final String password;
    
    public DataSource createDataSource() {
        // Create HikariCP or similar connection pool
    }
}
```

#### Service Usage Examples

**Starting the Service**
```bash
# Start the individual processing service
java -jar individual-processing-service.jar server individual-processing-service.yml

# Service will be available at:
# - Main API: http://localhost:8080/api/v1/
# - Admin/Health: http://localhost:8081/
# - Metrics: http://localhost:8081/metrics
```

**Client Service Integration Examples**

**Java Client (using Jersey Client)**
```java
public class IndividualProcessingClient {
    private final Client client;
    private final String baseUrl;
    
    public IndividualProcessingClient(String baseUrl) {
        this.client = ClientBuilder.newClient();
        this.baseUrl = baseUrl;
    }
    
    public AggregateIndividual getAggregateIndividual(long siteId, long individualId) {
        return client.target(baseUrl)
            .path("api/v1/individuals/{siteId}/{individualId}/aggregate")
            .resolveTemplate("siteId", siteId)
            .resolveTemplate("individualId", individualId)
            .request(MediaType.APPLICATION_JSON)
            .get(AggregateIndividual.class);
    }
    
    public BatchProcessingResponse batchProcessIndividuals(List<IndividualRequest> requests) {
        BatchProcessingRequest batchRequest = new BatchProcessingRequest(requests);
        
        return client.target(baseUrl)
            .path("api/v1/individuals/aggregate")
            .request(MediaType.APPLICATION_JSON)
            .post(Entity.json(batchRequest), BatchProcessingResponse.class);
    }
}
```

**cURL Examples**
```bash
# Get single individual
curl -X GET "http://individual-processor:8080/api/v1/individuals/1/123456/aggregate" \
     -H "Accept: application/json"

# Get searchable individual  
curl -X GET "http://individual-processor:8080/api/v1/individuals/1/123456/searchable" \
     -H "Accept: application/json"

# Batch process individuals
curl -X POST "http://individual-processor:8080/api/v1/individuals/aggregate" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d '{
       "individuals": [
         {"siteId": 1, "individualId": 123456},
         {"siteId": 1, "individualId": 123457}
       ],
       "options": {
         "includeExtendedRelationships": true,
         "maxRelationshipDepth": 2
       }
     }'

# Health check
curl -X GET "http://individual-processor:8081/healthcheck"
```

**Python Client Example**
```python
import requests
import json

class IndividualProcessingClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.session = requests.Session()
    
    def get_aggregate_individual(self, site_id, individual_id):
        url = f"{self.base_url}/api/v1/individuals/{site_id}/{individual_id}/aggregate"
        response = self.session.get(url)
        response.raise_for_status()
        return response.json()
    
    def batch_process_individuals(self, requests):
        url = f"{self.base_url}/api/v1/individuals/aggregate"
        payload = {
            "individuals": requests,
            "options": {
                "includeExtendedRelationships": True,
                "maxRelationshipDepth": 2
            }
        }
        response = self.session.post(url, json=payload)
        response.raise_for_status()
        return response.json()

# Usage
client = IndividualProcessingClient("http://individual-processor:8080")
result = client.get_aggregate_individual(1, 123456)
print(f"Individual: {result['firstName']} {result['lastName']}")
```

## Data Fetching Strategy

### Relationship Depth Analysis

For building accurate relationships, we need to fetch individuals at different relationship depths:

**Depth 0 (Target)**: The requested individual(s)
**Depth 1 (Immediate)**: Spouses, parents, children  
**Depth 2 (Extended)**: Siblings, grandparents, grandchildren, in-laws
**Depth 3+ (Optional)**: Extended family for comprehensive relationship building

### Optimized Fetching Algorithm

```java
public class RelationshipDataFetcher {
    
    public IndividualProcessingContext fetchMinimalContext(long siteId, Set<Long> targetIds) {
        Set<Long> allRequiredIds = new HashSet<>(targetIds);
        
        // Fetch immediate relationships (depth 1)
        for (Long targetId : targetIds) {
            allRequiredIds.addAll(getImmediateRelatives(siteId, targetId));
        }
        
        // Fetch extended relationships for sibling detection (depth 2)  
        Set<Long> depth2Ids = new HashSet<>();
        for (Long id : allRequiredIds) {
            depth2Ids.addAll(getSiblingsAndInLaws(siteId, id));
        }
        allRequiredIds.addAll(depth2Ids);
        
        return IndividualProcessingContext.buildMinimal(dbAccessor, siteId, treeId, 
                                                       targetIds, allRequiredIds);
    }
    
    private Set<Long> getImmediateRelatives(long siteId, long individualId) {
        // Query for spouses, parents, children
        // Based on Child_In_Family_ID, Adopted_Child_In_Family_ID, Foster_Child_In_Family_ID
        // and genealogy_family.Husband_ID/Wife_ID relationships
    }
}
```

## Performance Considerations

### Caching Strategy

**Multi-Level Caching Approach**
```java
public class CacheManager {
    private final Cache<String, AggregateIndividual> l1Cache; // In-memory
    private final RedisTemplate<String, AggregateIndividual> l2Cache; // Redis
    
    public AggregateIndividual get(String key, Class<AggregateIndividual> type) {
        // L1 Cache (In-Memory) - fastest
        AggregateIndividual result = l1Cache.getIfPresent(key);
        if (result != null) {
            metrics.counter("cache.l1.hits").inc();
            return result;
        }
        
        // L2 Cache (Redis) - shared across instances
        result = l2Cache.opsForValue().get(key);
        if (result != null) {
            l1Cache.put(key, result); // Promote to L1
            metrics.counter("cache.l2.hits").inc();
            return result;
        }
        
        metrics.counter("cache.misses").inc();
        return null;
    }
}
```

**Cache Key Strategy**
- **Individual Cache**: `agg:{siteId}:{individualId}:{includeExtended}:{version}`
- **Family Cache**: `family:{siteId}:{familyId}:{version}`
- **Relationship Cache**: `rel:{siteId}:{individualId}:{depth}:{version}`

**Cache Invalidation**
- **TTL-based**: 30 minutes for individuals, 1 hour for families
- **Version-based**: Increment version when individual data changes
- **Event-driven**: Invalidate cache when database updates occur

### Database Optimization

1. **Connection Pooling**: HikariCP with optimized settings
   ```yaml
   database:
     minSize: 8
     maxSize: 32
     maxWaitForConnection: 1s
     validationQuery: "SELECT 1"
   ```

2. **Prepared Statements**: Cache and reuse SQL queries
3. **Batch Fetching**: Group requests by site for efficient querying
4. **Read Replicas**: Distribute load across database replicas
5. **Database Indexes**: Ensure optimal indexes exist:
   ```sql
   -- Core relationship indexes
   CREATE INDEX idx_individual_site_id ON genealogy_individual(Site_ID, Individual_ID);
   CREATE INDEX idx_individual_child_family ON genealogy_individual(Site_ID, Family_Tree_ID, Child_In_Family_ID);
   CREATE INDEX idx_individual_adopted_family ON genealogy_individual(Site_ID, Family_Tree_ID, Adopted_Child_In_Family_ID);
   CREATE INDEX idx_family_husband ON genealogy_family(Site_ID, Family_Tree_ID, Husband_ID);
   CREATE INDEX idx_family_wife ON genealogy_family(Site_ID, Family_Tree_ID, Wife_ID);
   ```

### Memory Efficiency

1. **Minimal Context**: Only load necessary individuals and families
2. **Streaming Processing**: Process individuals without loading entire trees
3. **Object Reuse**: Reuse mapper and builder objects
4. **Memory Monitoring**: Track memory usage with JVM metrics

### Scalability

1. **Horizontal Scaling**: Deploy multiple service instances behind load balancer
2. **Database Load Distribution**: Use read replicas for query distribution
3. **Async Processing**: Use async processing for batch requests
4. **Circuit Breakers**: Implement circuit breakers for database resilience

## Migration and Compatibility

### Backward Compatibility
- Existing batch processing mode remains unchanged
- Same output format (`AggregateIndividual`, `SearchableIndividual`)
- Existing APIs and downstream consumers unaffected

### Gradual Migration
1. **Phase 1**: Implement individual mode alongside batch mode
2. **Phase 2**: Test with small-scale individual processing
3. **Phase 3**: Optimize and scale individual mode
4. **Phase 4**: Consider hybrid approaches for different use cases

## Use Cases and Benefits

### Primary Service Use Cases

1. **Matching Service Integration**
   - On-demand individual processing for matching algorithms
   - Real-time relationship data for match scoring
   - Individual updates without full tree reprocessing

2. **Search Service Enhancement** 
   - Real-time SearchableIndividual generation for search indexing
   - Individual profile updates for search results
   - Dynamic relationship data for search filtering

3. **User Profile Services**
   - Real-time individual data for user profile pages
   - Relationship data for family tree visualization
   - Individual updates when users edit their profiles

4. **API Gateway Services**
   - External API endpoints for individual data
   - Third-party integrations requiring individual genealogy data
   - Mobile app backends needing individual processing

5. **Analytics and Reporting**
   - Individual-level analytics without full tree processing
   - Real-time reporting on individual data changes
   - Targeted data extraction for business intelligence

6. **Development and Testing**
   - Individual-level testing without processing entire trees
   - Development environment data setup
   - Debugging specific individual relationship issues

### Service Benefits

1. **Sub-Second Response Times**: Process individuals in 100-500ms vs. hours for full trees
2. **High Availability**: Service-based architecture with health checks and monitoring
3. **Horizontal Scalability**: Deploy multiple instances behind load balancers
4. **Real-time Capability**: Enable real-time individual updates across all services
5. **Resource Efficiency**: Minimal memory and CPU usage per request
6. **Service Isolation**: Individual processing failures don't affect other services
7. **API Standardization**: Consistent REST API for all individual processing needs
8. **Caching Benefits**: Intelligent caching reduces database load and improves performance

### Example Service Integration Scenarios

**Scenario 1: User Profile Update**
```
User updates profile → User Service → Individual Processing Service → Updated AggregateIndividual → Search Service (reindex)
```

**Scenario 2: Matching Algorithm**
```
Matching Service → Individual Processing Service (batch request) → AggregateIndividuals → Relationship-aware matching
```

**Scenario 3: Mobile App**
```
Mobile App → API Gateway → Individual Processing Service → SearchableIndividual → Family tree display
```

## Implementation Timeline

### Phase 1: Foundation (4-6 weeks)
- [ ] Implement `IndividualDatabaseAccessor` with basic queries
- [ ] Create `IndividualProcessingContext` for minimal context building
- [ ] Develop `IndividualAggregateProcessor` core logic
- [ ] Add database configuration management

### Phase 2: Integration (3-4 weeks)  
- [ ] Modify `TreeProcessorRunner` to support individual mode
- [ ] Implement `IndividualProcessorPipe` for pipeline integration
- [ ] Create `IndividualRelativesBuilder` for targeted relationship building
- [ ] Add command-line interface and configuration

### Phase 3: Testing and Optimization (3-4 weeks)
- [ ] Unit tests for all new components
- [ ] Integration tests with real database
- [ ] Performance testing and optimization
- [ ] Memory usage analysis and optimization

### Phase 4: Production Readiness (2-3 weeks)
- [ ] Error handling and resilience
- [ ] Monitoring and metrics
- [ ] Documentation and examples
- [ ] Production deployment preparation

## Conclusion

This proposal provides a comprehensive approach to creating a new **Individual Processing Service** that enables real-time, on-demand processing of genealogy individuals. The service-based architecture provides significant advantages over the existing batch processing approach for targeted use cases.

### Key Innovations

1. **Service-Oriented Architecture**: RESTful service that other services can easily integrate with
2. **Selective Data Fetching**: Intelligently determines minimal set of related individuals needed for accurate relationship building  
3. **Database-Direct Access**: Bypasses Avro file processing for real-time database queries
4. **Multi-Level Caching**: Optimized caching strategy for high performance and scalability
5. **Batch Processing Support**: Efficiently handles both single and batch individual requests

### Strategic Benefits

1. **Microservices Enablement**: Enables other services to process individuals without dependency on batch processing
2. **Real-Time Capability**: Sub-second response times enable real-time user experiences
3. **Resource Optimization**: Minimal resource usage compared to full tree processing
4. **Service Isolation**: Failures in individual processing don't affect other system components
5. **API Standardization**: Consistent interface for all individual processing needs across the organization

### Implementation Feasibility

The proposed solution is highly feasible because it:
- Reuses existing relationship building logic (`RelativesBuilder`, `SearchableIndividualMapper`)
- Maintains compatibility with existing data formats (`AggregateIndividual`, `SearchableIndividual`)
- Leverages proven database infrastructure and connection patterns
- Uses standard service frameworks (Dropwizard) already in use at MyHeritage

This service will unlock new possibilities for real-time genealogy applications, improve user experiences through faster individual data access, and provide a foundation for future microservices that need individual-level genealogy processing capabilities.
