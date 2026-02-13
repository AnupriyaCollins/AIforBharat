# Design Document: UDAN MRO Control Tower

## System Overview

The UDAN MRO Control Tower is an AWS-native, AI-powered decision-support platform designed to reduce Aircraft on Ground (AOG) events and improve maintenance turnaround times for India's regional connectivity operations. The system integrates real-time flight operations data, open aircraft health datasets, and synthetic maintenance logs to provide predictive maintenance insights, risk-based dispatch recommendations, and natural language query capabilities through an agentic AI copilot.

### Core Capabilities

1. **Real-Time Risk Assessment**: Continuous evaluation of aircraft maintenance risk based on operational patterns, health indicators, and maintenance history
2. **Predictive Maintenance**: ML-based failure prediction for critical aircraft components with remaining useful life estimates
3. **Agentic AI Copilot**: Bedrock-powered natural language interface for maintenance queries with tool invocation and reasoning capabilities
4. **UDAN-Aware Optimization**: Dispatch and spares positioning recommendations that account for Tier-2/3 airport constraints
5. **Explainable Decisions**: Complete audit trails and human-readable explanations for all AI-generated recommendations

### Key Design Principles

**Modularity**: The architecture follows a layered design with clear separation between data ingestion, analytics, AI orchestration, and presentation layers. Each component can be developed, tested, and scaled independently.

**Explainability**: Every AI decision includes traceable reasoning. Risk scores decompose into contributing factors, predictive models provide feature importance, and the AI copilot cites sources for all responses.

**Auditability**: All system actions are logged with immutable audit trails stored in S3. Every recommendation includes timestamp, input data snapshot, model version, and reasoning chain.

**Serverless-First**: The system leverages AWS Lambda, Step Functions, and managed services to minimize operational overhead and enable elastic scaling.

**Data Ethics**: Clear provenance tracking distinguishes between real public data, open datasets, and synthetic data. No proprietary or restricted information is used.

**Fail-Safe Design**: The system is advisory-only. All recommendations require human validation, and system failures default to conservative maintenance decisions.

## Architecture Diagram (Textual Description)

### Component Overview

The system consists of five primary layers:


**Layer 1: Data Ingestion**
- Flight Operations Ingestion Service (Lambda): Polls public flight APIs, processes schedules and delays
- Maintenance Log Processor (Lambda): Parses ATA-structured maintenance records
- Aircraft Health Data Loader (Lambda): Loads open datasets (NASA C-MAPSS) into S3
- Raw Data Lake (S3): Stores unprocessed data with versioning
- Curated Data Store (DynamoDB): Stores validated, structured data for analytics

**Layer 2: AI & Analytics**
- Risk Scoring Engine (Lambda): Calculates real-time maintenance risk scores
- Predictive Maintenance Model (SageMaker Endpoint): Provides component failure predictions
- Feature Engineering Pipeline (Step Functions): Orchestrates data preparation for ML models
- Model Registry (SageMaker): Manages model versions and deployment

**Layer 3: Agentic AI & Orchestration**
- Maintenance Copilot (Bedrock Agent): Natural language interface with tool invocation
- Tool Functions (Lambda): Executable functions for data retrieval, risk calculation, and recommendations
- Orchestration Workflow (Step Functions): Coordinates multi-step decision processes
- Decision Trace Store (DynamoDB): Captures reasoning chains and tool invocations

**Layer 4: Knowledge & RAG**
- Maintenance Knowledge Base (S3): Stores maintenance manuals, procedures, and best practices
- Vector Store (OpenSearch Serverless): Enables semantic search over maintenance knowledge
- RAG Pipeline (Lambda + Bedrock): Retrieves relevant context for copilot responses
- Embedding Service (Bedrock): Generates embeddings for knowledge documents

**Layer 5: Application & Visualization**
- Dashboard API (API Gateway + Lambda): RESTful API for frontend interactions
- Web Dashboard (S3 + CloudFront): React-based visualization interface
- Real-Time Updates (WebSocket API): Pushes risk score updates to connected clients
- Report Generator (Lambda): Creates exportable PDF/CSV reports

### Data Flow

1. **Ingestion Flow**: Flight data → Lambda → Validation → DynamoDB → EventBridge → Risk Scoring
2. **Prediction Flow**: Aircraft health data → Feature engineering → SageMaker inference → Risk scoring
3. **Query Flow**: User question → Bedrock Agent → Tool selection → Lambda execution → Response generation
4. **Audit Flow**: All actions → CloudWatch Logs → S3 archive → Athena queryable

### Integration Points

- **EventBridge**: Decouples components through event-driven architecture
- **API Gateway**: Provides unified REST and WebSocket endpoints
- **IAM**: Enforces least-privilege access across all services
- **KMS**: Manages encryption keys for data at rest

## Data Ingestion Layer

### Flight Operations Data Ingestion

**Data Sources**:
- Public flight tracking APIs (e.g., OpenSky Network, ADS-B Exchange)
- UDAN route schedules from public aviation databases
- Delay and cancellation data from aggregated sources

**Ingestion Architecture**:

The Flight Operations Ingestion Service runs as a scheduled Lambda function (EventBridge cron: every 15 minutes) that:

1. **Fetches** flight data from configured public APIs using HTTP clients
2. **Validates** data completeness (required fields: flight number, aircraft registration, departure/arrival times, route)
3. **Enriches** with UDAN route metadata (airport tier classification, MRO facility availability)
4. **Transforms** into standardized schema with ISO 8601 timestamps and ICAO airport codes
5. **Stores** raw JSON in S3 (`s3://udan-mro-raw/flight-ops/YYYY/MM/DD/HH/batch-{uuid}.json`)
6. **Writes** validated records to DynamoDB table `FlightOperations` with composite key (aircraft_registration, timestamp)
7. **Publishes** `FlightDataUpdated` event to EventBridge for downstream processing

**Error Handling**:
- API failures trigger exponential backoff retry (3 attempts, max 60s delay)
- Malformed data is logged to CloudWatch and stored in S3 dead-letter bucket
- Missing critical fields result in record rejection with detailed error logging

### UDAN Route and Airport Metadata

**Metadata Schema**:


```json
{
  "airport_code": "ICAO code",
  "airport_name": "Full name",
  "tier": "1 | 2 | 3",
  "mro_capabilities": {
    "line_maintenance": true,
    "heavy_maintenance": false,
    "avionics_repair": false,
    "engine_shop": false
  },
  "spares_inventory": ["component_codes"],
  "certified_personnel_count": 5,
  "nearest_major_hub": "ICAO code",
  "distance_to_hub_km": 450
}
```

This metadata is stored in DynamoDB table `AirportMetadata` and loaded during system initialization from a curated JSON file. Updates are manual through an admin API.

**UDAN Route Schema**:

```json
{
  "route_id": "unique identifier",
  "origin": "ICAO code",
  "destination": "ICAO code",
  "aircraft_types": ["ATR-72", "Dash-8-Q400"],
  "frequency_per_week": 7,
  "average_utilization_hours": 4.5,
  "criticality": "high | medium | low"
}
```

Routes are stored in DynamoDB table `UDANRoutes` and used by the dispatch optimizer to assess operational impact of maintenance decisions.

### Storage Strategy

**Raw Data Lake (S3)**:
- **Purpose**: Immutable archive of all ingested data for audit and reprocessing
- **Structure**: Partitioned by data type, year, month, day, hour
- **Retention**: 2 years with lifecycle policy to Glacier after 90 days
- **Versioning**: Enabled for compliance and rollback capability
- **Encryption**: SSE-KMS with customer-managed key

**Curated Data Store (DynamoDB)**:
- **Purpose**: Validated, queryable data for real-time analytics
- **Tables**:
  - `FlightOperations`: Partition key = aircraft_registration, Sort key = timestamp
  - `MaintenanceLogs`: Partition key = aircraft_registration, Sort key = event_timestamp
  - `AircraftHealth`: Partition key = aircraft_registration, Sort key = sensor_reading_timestamp
  - `AirportMetadata`: Partition key = airport_code
  - `UDANRoutes`: Partition key = route_id
- **Indexes**: GSI on aircraft_type, route_id, risk_score for efficient queries
- **TTL**: FlightOperations records expire after 90 days (historical data in S3)
- **Point-in-Time Recovery**: Enabled for data protection

**Data Quality Checks**:
- Schema validation using JSON Schema before DynamoDB writes
- Referential integrity checks (aircraft registration exists, airport codes valid)
- Timestamp sanity checks (no future dates, reasonable historical range)
- Duplicate detection using idempotency keys

## AI & Analytics Layer

### Predictive Maintenance Model

**Model Architecture**:

The predictive maintenance model is a gradient boosting ensemble (XGBoost) trained on NASA C-MAPSS turbofan degradation dataset, adapted for regional aircraft components.

**Inputs**:
- **Sensor Readings**: Engine parameters (temperature, pressure, vibration), system health indicators
- **Operational Context**: Flight hours since last maintenance, cycles, average flight duration
- **Maintenance History**: Time since last component replacement, cumulative maintenance events
- **Environmental Factors**: Average operating altitude, climate conditions at base airport

**Outputs**:
- **Failure Probability**: 0-1 probability of component failure within next 30/60/90 days
- **Remaining Useful Life (RUL)**: Estimated flight hours until maintenance required
- **Confidence Interval**: 95% confidence bounds on RUL estimate
- **Feature Importance**: SHAP values indicating which inputs most influenced the prediction

**Training Pipeline**:

1. **Data Preparation** (Step Functions workflow):
   - Extract features from aircraft health data and maintenance logs
   - Normalize sensor readings using historical statistics
   - Create sliding time windows (7-day, 30-day aggregates)
   - Split into train/validation/test sets (70/15/15)

2. **Model Training** (SageMaker Training Job):
   - XGBoost algorithm with hyperparameter tuning (Bayesian optimization)
   - Objective: Minimize false negatives (missed failures) while controlling false positives
   - Evaluation metrics: Precision, Recall, F1-score, AUC-ROC
   - Early stopping based on validation loss

3. **Model Evaluation**:
   - Test set performance must achieve 80%+ recall for high-risk predictions
   - Calibration check: predicted probabilities match observed failure rates
   - Fairness check: performance consistent across aircraft types

4. **Model Registration**:
   - Approved models registered in SageMaker Model Registry with metadata
   - Version tagging includes training date, dataset version, performance metrics

**Deployment Strategy**:

- **Endpoint**: SageMaker real-time endpoint with auto-scaling (min 1, max 5 instances)
- **Instance Type**: ml.m5.xlarge (cost-optimized for hackathon, production would use ml.c5)
- **Invocation**: Lambda function calls endpoint with batch inference for efficiency
- **Caching**: Recent predictions cached in DynamoDB (TTL 1 hour) to reduce inference costs
- **Monitoring**: CloudWatch metrics track latency, error rate, and model drift

**Limitations**:

- **Training Data**: Model trained on turbofan data, not all regional aircraft components
- **Generalization**: Performance may degrade for aircraft types not in training data
- **Sensor Availability**: Predictions require minimum sensor coverage (80% of expected readings)
- **Cold Start**: New aircraft require 30 days of operational data for reliable predictions
- **External Factors**: Model does not account for pilot behavior, weather extremes, or operational anomalies

### Risk Scoring Logic

The Risk Scoring Engine combines multiple signals into a unified 0-100 risk score:

**Scoring Components**:

1. **Predictive Risk (40% weight)**:
   - Failure probability from ML model
   - Scaled: 0.8+ probability → 100 points, <0.2 probability → 0 points

2. **Operational Risk (30% weight)**:
   - Flight hours since last maintenance / recommended interval
   - Recent delay frequency (maintenance-related delays in last 7 days)
   - Upcoming high-utilization routes

3. **Historical Risk (20% weight)**:
   - Recurrent defects (same ATA chapter issues in last 90 days)
   - Component age relative to manufacturer-recommended life limits
   - Maintenance backlog (deferred items)

4. **Contextual Risk (10% weight)**:
   - Destination airport MRO capability (low capability → higher risk)
   - Spares availability at planned destinations
   - Weather or operational constraints

**Calculation**:

```
risk_score = (
    0.40 * predictive_risk +
    0.30 * operational_risk +
    0.20 * historical_risk +
    0.10 * contextual_risk
)
```

**Thresholds**:
- **Low Risk**: 0-33 (routine monitoring)
- **Medium Risk**: 34-66 (increased inspection frequency)
- **High Risk**: 67-100 (immediate maintenance review required)

**Explainability**:

Each risk score includes a breakdown showing:
- Individual component scores and weights
- Top 3 contributing factors with descriptions
- Recommended actions based on risk level
- Confidence level (based on data completeness)

### Model Deployment Strategy

**CI/CD Pipeline**:

1. **Model Training**: Triggered manually or on schedule (weekly retraining)
2. **Validation Gate**: Automated tests verify model performance meets thresholds
3. **Staging Deployment**: Model deployed to staging endpoint for integration testing
4. **A/B Testing**: New model serves 10% of traffic, compared against current production model
5. **Production Promotion**: If new model outperforms, gradual rollout to 100% traffic
6. **Rollback**: Automated rollback if error rate exceeds threshold

**Monitoring**:

- **Model Drift**: Compare prediction distribution against training data distribution
- **Performance Degradation**: Track precision/recall on labeled production data
- **Data Quality**: Monitor input feature distributions for anomalies
- **Business Metrics**: Track correlation between predictions and actual AOG events

## Agentic AI & Orchestration

### Role of the Bedrock-Powered Maintenance Copilot

The Maintenance Copilot is an agentic AI system built on Amazon Bedrock Agents that provides natural language interaction for maintenance decision support. Unlike traditional chatbots, the copilot can:


**Capabilities**:

1. **Query Aircraft Status**: "What's the maintenance risk for VT-ABC?"
2. **Explain Predictions**: "Why is VT-XYZ flagged as high risk?"
3. **Recommend Actions**: "Should I dispatch VT-DEF to Guwahati given its current status?"
4. **Retrieve History**: "Show me all engine-related issues for VT-GHI in the last 60 days"
5. **Optimize Spares**: "Which spares should I position at Agra airport this week?"
6. **Compare Scenarios**: "What's the risk difference between dispatching now vs. after line maintenance?"

**Architecture**:

The copilot uses Bedrock Agents with Claude 3 Sonnet as the foundation model. The agent is configured with:

- **Instructions**: System prompt defining role, capabilities, and response format
- **Action Groups**: Collections of tools the agent can invoke
- **Knowledge Bases**: RAG-enabled access to maintenance documentation

**Agent Configuration**:

```yaml
agent_name: UDAN-MRO-Copilot
foundation_model: anthropic.claude-3-sonnet-20240229-v1:0
instructions: |
  You are an AI maintenance advisor for UDAN regional aircraft operations.
  Your role is to help operations teams and MRO planners make informed decisions.
  
  Always:
  - Cite data sources and confidence levels
  - Explain your reasoning step-by-step
  - Acknowledge limitations and uncertainties
  - Recommend human validation for critical decisions
  
  Never:
  - Make airworthiness certification claims
  - Override DGCA regulations
  - Provide advice outside your knowledge base
  
action_groups:
  - name: aircraft_status
    description: Retrieve current aircraft health and risk scores
  - name: maintenance_history
    description: Query historical maintenance events
  - name: dispatch_advisor
    description: Provide dispatch recommendations
  - name: spares_optimizer
    description: Recommend spares positioning
```

### Tool Invocation via Lambda

Each action group maps to a Lambda function that executes the requested operation:

**Tool: aircraft_status**

```python
def get_aircraft_status(aircraft_registration: str) -> dict:
    # Query DynamoDB for latest risk score
    risk_data = dynamodb.get_item(
        TableName='RiskScores',
        Key={'aircraft_registration': aircraft_registration}
    )
    
    # Query SageMaker endpoint for latest prediction
    prediction = sagemaker_runtime.invoke_endpoint(
        EndpointName='predictive-maintenance-model',
        Body=json.dumps({'aircraft_id': aircraft_registration})
    )
    
    # Combine and format response
    return {
        'aircraft_registration': aircraft_registration,
        'risk_score': risk_data['risk_score'],
        'risk_level': risk_data['risk_level'],
        'contributing_factors': risk_data['factors'],
        'failure_probability_30d': prediction['failure_prob_30d'],
        'remaining_useful_life_hours': prediction['rul_hours'],
        'confidence': prediction['confidence'],
        'last_updated': risk_data['timestamp']
    }
```

**Tool: dispatch_advisor**

```python
def get_dispatch_recommendation(
    aircraft_registration: str,
    destination_airport: str,
    flight_duration_hours: float
) -> dict:
    # Get current risk score
    risk = get_aircraft_status(aircraft_registration)
    
    # Get destination MRO capabilities
    airport = dynamodb.get_item(
        TableName='AirportMetadata',
        Key={'airport_code': destination_airport}
    )
    
    # Decision logic
    if risk['risk_score'] >= 67:
        recommendation = 'NO_DISPATCH'
        reasoning = 'High maintenance risk requires immediate inspection'
    elif risk['risk_score'] >= 34 and not airport['mro_capabilities']['line_maintenance']:
        recommendation = 'DISPATCH_WITH_CAUTION'
        reasoning = 'Medium risk and limited MRO at destination'
    else:
        recommendation = 'DISPATCH_APPROVED'
        reasoning = 'Risk within acceptable limits'
    
    return {
        'recommendation': recommendation,
        'reasoning': reasoning,
        'risk_score': risk['risk_score'],
        'destination_mro_capability': airport['mro_capabilities'],
        'alternate_airports': find_nearby_airports_with_mro(destination_airport),
        'confidence': 'high' if risk['confidence'] > 0.8 else 'medium'
    }
```

**Tool Invocation Flow**:

1. User submits query to Bedrock Agent via API Gateway
2. Agent parses intent and selects appropriate action group
3. Agent generates tool invocation request with parameters
4. Bedrock invokes Lambda function via function URL
5. Lambda executes business logic, queries data stores
6. Lambda returns structured response to Bedrock
7. Agent synthesizes natural language response incorporating tool results
8. Response returned to user with citations

**Error Handling**:

- Lambda timeouts (30s) trigger fallback response: "Unable to retrieve data, please try again"
- Invalid parameters return error with guidance: "Aircraft registration must be in format VT-XXX"
- Data not found returns informative message: "No maintenance history found for this aircraft"

### Workflow Orchestration via Step Functions

Complex multi-step processes are orchestrated using AWS Step Functions:

**Workflow: Daily Risk Assessment**

```json
{
  "Comment": "Daily risk assessment for all active aircraft",
  "StartAt": "GetActiveAircraft",
  "States": {
    "GetActiveAircraft": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:GetActiveAircraft",
      "Next": "ProcessAircraftInParallel"
    },
    "ProcessAircraftInParallel": {
      "Type": "Map",
      "ItemsPath": "$.aircraft_list",
      "MaxConcurrency": 10,
      "Iterator": {
        "StartAt": "FetchAircraftData",
        "States": {
          "FetchAircraftData": {
            "Type": "Parallel",
            "Branches": [
              {
                "StartAt": "GetFlightHistory",
                "States": {
                  "GetFlightHistory": {
                    "Type": "Task",
                    "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:GetFlightHistory",
                    "End": true
                  }
                }
              },
              {
                "StartAt": "GetMaintenanceHistory",
                "States": {
                  "GetMaintenanceHistory": {
                    "Type": "Task",
                    "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:GetMaintenanceHistory",
                    "End": true
                  }
                }
              },
              {
                "StartAt": "GetHealthData",
                "States": {
                  "GetHealthData": {
                    "Type": "Task",
                    "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:GetHealthData",
                    "End": true
                  }
                }
              }
            ],
            "Next": "CalculateRiskScore"
          },
          "CalculateRiskScore": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:CalculateRiskScore",
            "Next": "CheckRiskThreshold"
          },
          "CheckRiskThreshold": {
            "Type": "Choice",
            "Choices": [
              {
                "Variable": "$.risk_score",
                "NumericGreaterThanEquals": 67,
                "Next": "GenerateHighRiskAlert"
              }
            ],
            "Default": "StoreRiskScore"
          },
          "GenerateHighRiskAlert": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:SendAlert",
            "Next": "StoreRiskScore"
          },
          "StoreRiskScore": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:StoreRiskScore",
            "End": true
          }
        }
      },
      "Next": "GenerateDailyReport"
    },
    "GenerateDailyReport": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:GenerateDailyReport",
      "End": true
    }
  }
}
```

This workflow runs daily at 6 AM IST (EventBridge schedule) and processes all aircraft in parallel for efficiency.

**Workflow: Maintenance Decision Process**

For complex decisions requiring multiple data sources and reasoning steps:

1. **Gather Context**: Fetch aircraft status, route information, weather, spares availability
2. **Assess Risk**: Calculate current risk score and predictive maintenance forecast
3. **Evaluate Options**: Compare dispatch now vs. defer for maintenance
4. **Generate Recommendation**: Synthesize data into actionable recommendation
5. **Capture Decision Trace**: Log all inputs, intermediate steps, and final output

### Explainability and Decision Traces

**Decision Trace Schema**:

```json
{
  "trace_id": "uuid",
  "timestamp": "ISO 8601",
  "user_id": "user identifier",
  "query": "original user question",
  "agent_reasoning": [
    {
      "step": 1,
      "thought": "I need to check the aircraft's current risk score",
      "action": "invoke_tool",
      "tool": "aircraft_status",
      "parameters": {"aircraft_registration": "VT-ABC"},
      "result": {"risk_score": 72, "risk_level": "high"}
    },
    {
      "step": 2,
      "thought": "High risk score requires understanding contributing factors",
      "action": "analyze_factors",
      "factors": [
        {"factor": "engine_vibration_trend", "importance": 0.35},
        {"factor": "hours_since_maintenance", "importance": 0.28}
      ]
    }
  ],
  "final_response": "Aircraft VT-ABC has a high maintenance risk (score: 72)...",
  "confidence": 0.85,
  "data_sources": [
    "DynamoDB:RiskScores",
    "SageMaker:predictive-maintenance-model"
  ],
  "model_version": "v1.2.3"
}
```

**Storage and Retrieval**:

- Decision traces stored in DynamoDB table `DecisionTraces`
- Partition key: trace_id, GSI on user_id and timestamp
- Queryable via dashboard for audit and review
- Retention: 2 years, then archived to S3

**Explainability Features**:

1. **Reasoning Chain**: Step-by-step breakdown of agent's thought process
2. **Data Provenance**: Every fact cited with source and timestamp
3. **Confidence Scoring**: Explicit uncertainty quantification
4. **Alternative Scenarios**: "What if" analysis showing impact of different inputs
5. **Human-Readable Summaries**: Technical details abstracted for non-experts

## Knowledge & RAG Layer

### Maintenance Knowledge Sources

The system maintains a curated knowledge base of maintenance-related information:

**Knowledge Categories**:

1. **Maintenance Manuals**: Aircraft maintenance manuals (AMM) for supported aircraft types
2. **Service Bulletins**: Manufacturer service bulletins and airworthiness directives
3. **Best Practices**: Industry best practices for regional aircraft maintenance
4. **Troubleshooting Guides**: Common fault diagnosis and resolution procedures
5. **Regulatory Guidance**: DGCA regulations and advisory circulars (public domain)
6. **Historical Case Studies**: Anonymized maintenance case studies and lessons learned

**Data Sources**:

- Public domain maintenance documentation
- Open-source aviation safety databases (NASA ASRS, EASA safety publications)
- Synthetically generated troubleshooting guides based on industry patterns
- Curated excerpts from publicly available regulatory documents

**Document Processing**:


1. **Ingestion**: Documents uploaded to S3 bucket `udan-mro-knowledge`
2. **Chunking**: Lambda function splits documents into semantic chunks (500-1000 tokens)
3. **Metadata Extraction**: Extract document type, aircraft type, ATA chapter, publication date
4. **Embedding**: Bedrock Embeddings (Titan Embeddings G1) generates vector representations
5. **Indexing**: Vectors stored in OpenSearch Serverless with metadata

### Vector Store Strategy

**OpenSearch Serverless Configuration**:

- **Collection**: `maintenance-knowledge`
- **Index**: `knowledge-chunks`
- **Dimensions**: 1536 (Titan Embeddings G1 output size)
- **Similarity Metric**: Cosine similarity
- **Capacity**: 2 OCU (OpenSearch Compute Units) for hackathon MVP

**Index Schema**:

```json
{
  "mappings": {
    "properties": {
      "chunk_id": {"type": "keyword"},
      "document_id": {"type": "keyword"},
      "document_title": {"type": "text"},
      "document_type": {"type": "keyword"},
      "aircraft_type": {"type": "keyword"},
      "ata_chapter": {"type": "keyword"},
      "content": {"type": "text"},
      "embedding": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "name": "hnsw",
          "engine": "nmslib",
          "parameters": {"ef_construction": 128, "m": 16}
        }
      },
      "metadata": {"type": "object"}
    }
  }
}
```

**Search Strategy**:

- **Hybrid Search**: Combine vector similarity with keyword filtering
- **Filters**: Apply aircraft_type and ata_chapter filters before vector search
- **Re-ranking**: Use Bedrock Rerank API to improve relevance of top results
- **Top-K**: Retrieve top 5 chunks, expand to include adjacent chunks for context

### Query and Retrieval Flow

**RAG Pipeline**:

1. **Query Reception**: User asks question via copilot
2. **Query Understanding**: Bedrock Agent extracts intent and entities (aircraft type, component, issue)
3. **Query Expansion**: Generate alternative phrasings and related terms
4. **Vector Search**: Convert query to embedding, search OpenSearch
5. **Filtering**: Apply metadata filters (aircraft type, ATA chapter)
6. **Retrieval**: Fetch top 5 most relevant chunks
7. **Context Assembly**: Combine chunks with conversation history
8. **Response Generation**: Bedrock generates response grounded in retrieved context
9. **Citation**: Include source document references in response

**Example Flow**:

```
User Query: "What's the procedure for troubleshooting high engine vibration on ATR-72?"

1. Query Embedding: [0.123, -0.456, ...] (1536 dimensions)

2. OpenSearch Query:
   {
     "query": {
       "bool": {
         "must": [
           {"knn": {"embedding": {"vector": [...], "k": 10}}}
         ],
         "filter": [
           {"term": {"aircraft_type": "ATR-72"}},
           {"term": {"ata_chapter": "71"}}  // Powerplant
         ]
       }
     }
   }

3. Retrieved Chunks:
   - "ATR-72 Engine Vibration Troubleshooting Guide" (score: 0.89)
   - "Vibration Monitoring Procedures" (score: 0.85)
   - "Engine Health Monitoring System" (score: 0.82)

4. Response Generation:
   "For high engine vibration on ATR-72, follow these steps:
   1. Check vibration sensor readings in EHMS
   2. Inspect engine mounts for wear or damage
   3. Verify propeller balance...
   
   Source: ATR-72 Maintenance Manual, Chapter 71-00-00"
```

**Caching**:

- Frequently asked questions cached in ElastiCache (Redis)
- Cache key: hash of query + filters
- TTL: 24 hours
- Cache hit rate target: 40%+ for common queries

## Application & Visualization Layer

### Dashboards

The web dashboard provides real-time visibility into fleet health and maintenance status.

**Dashboard Components**:

1. **Fleet Overview**:
   - Map view showing aircraft locations with risk score color coding
   - Summary statistics: total aircraft, high-risk count, AOG count
   - Trend charts: risk score distribution over time

2. **Aircraft Detail View**:
   - Current risk score with contributing factors breakdown
   - Predictive maintenance forecast (30/60/90 day failure probabilities)
   - Recent flight history and maintenance events
   - Upcoming scheduled maintenance

3. **Route Reliability View**:
   - UDAN route performance metrics (on-time %, cancellation rate)
   - Maintenance-related delay analysis by route
   - Airport MRO capability matrix

4. **Spares Optimization View**:
   - Current spares inventory by location
   - Predicted spares demand (next 30 days)
   - Recommended spares positioning actions

5. **Alerts and Notifications**:
   - Real-time alerts for high-risk aircraft
   - Maintenance due reminders
   - Spares shortage warnings

**Technology Stack**:

- **Frontend**: React 18 with TypeScript
- **UI Framework**: Material-UI for consistent design
- **Charts**: Recharts for data visualization
- **State Management**: React Query for server state, Zustand for client state
- **Real-Time Updates**: WebSocket connection for live risk score updates
- **Hosting**: S3 static hosting with CloudFront CDN

**API Integration**:

Dashboard communicates with backend via REST API (API Gateway):

- `GET /api/fleet/status` - Get all aircraft status
- `GET /api/aircraft/{registration}` - Get specific aircraft details
- `GET /api/risk-scores` - Get current risk scores
- `GET /api/maintenance-history/{registration}` - Get maintenance history
- `POST /api/copilot/query` - Submit query to AI copilot
- `GET /api/reports/generate` - Generate exportable report

WebSocket API (`wss://api.udan-mro.example.com`) pushes real-time updates:

- Risk score changes
- New maintenance events
- Alert notifications

### User Interaction Patterns

**Typical User Workflows**:

1. **Morning Fleet Review** (Operations Team):
   - Log in to dashboard
   - Review overnight risk score changes
   - Check high-risk aircraft alerts
   - Query copilot: "Any aircraft requiring immediate attention?"
   - Review dispatch recommendations for today's flights

2. **Maintenance Planning** (MRO Planner):
   - Review predictive maintenance forecasts
   - Identify aircraft approaching maintenance thresholds
   - Query copilot: "Which aircraft need maintenance in the next 2 weeks?"
   - Check spares availability for planned maintenance
   - Schedule maintenance activities

3. **Dispatch Decision** (Operations Team):
   - Receive alert: Aircraft VT-ABC has elevated risk score
   - View aircraft detail page for risk breakdown
   - Query copilot: "Should I dispatch VT-ABC to Guwahati?"
   - Review recommendation and reasoning
   - Make final dispatch decision (approve/defer)
   - Log decision in system

4. **Incident Investigation** (MRO Planner):
   - AOG event occurs
   - Review aircraft maintenance history
   - Query copilot: "What were the warning signs for this failure?"
   - Examine decision traces leading up to event
   - Generate incident report

### Decision Review and Override

**Human-in-the-Loop Design**:

All AI recommendations are advisory and require human confirmation:

1. **Recommendation Display**: System presents recommendation with confidence level
2. **Reasoning Transparency**: User can expand to see full reasoning chain
3. **Override Option**: User can accept, reject, or modify recommendation
4. **Justification Required**: Overrides require user to provide justification
5. **Audit Trail**: All decisions (accepted/rejected) logged with user ID and timestamp

**Override Interface**:

```
┌─────────────────────────────────────────────────┐
│ Dispatch Recommendation: NO DISPATCH            │
│ Confidence: High (0.87)                         │
│                                                 │
│ Reasoning:                                      │
│ • Risk score: 72 (High)                        │
│ • Engine vibration trending upward             │
│ • Destination has limited MRO capability       │
│                                                 │
│ [View Full Analysis] [View Decision Trace]     │
│                                                 │
│ ┌─────────────────────────────────────────┐   │
│ │ ○ Accept Recommendation                  │   │
│ │ ○ Override - Dispatch Anyway             │   │
│ │ ○ Defer Decision                         │   │
│ └─────────────────────────────────────────┘   │
│                                                 │
│ If overriding, provide justification:          │
│ ┌─────────────────────────────────────────┐   │
│ │ [Text area for justification]            │   │
│ └─────────────────────────────────────────┘   │
│                                                 │
│ [Cancel] [Submit Decision]                     │
└─────────────────────────────────────────────────┘
```

**Decision Logging**:

```json
{
  "decision_id": "uuid",
  "timestamp": "ISO 8601",
  "user_id": "user identifier",
  "aircraft_registration": "VT-ABC",
  "recommendation": "NO_DISPATCH",
  "user_decision": "OVERRIDE_DISPATCH",
  "justification": "Maintenance team inspected engine, vibration within limits",
  "risk_score_at_decision": 72,
  "outcome": "to_be_determined"
}
```

Outcomes are tracked post-decision to measure recommendation accuracy and identify improvement opportunities.

## Security, Governance, and Ethics

### IAM Strategy

**Principle of Least Privilege**:

Every component has minimal permissions required for its function.

**IAM Roles**:

1. **FlightDataIngestionRole**:
   - S3: PutObject on raw data bucket
   - DynamoDB: PutItem on FlightOperations table
   - EventBridge: PutEvents
   - CloudWatch: PutLogEvents

2. **RiskScoringRole**:
   - DynamoDB: GetItem, Query on all tables
   - SageMaker: InvokeEndpoint on predictive model
   - DynamoDB: PutItem on RiskScores table

3. **CopilotToolRole**:
   - DynamoDB: GetItem, Query (read-only)
   - SageMaker: InvokeEndpoint (read-only)
   - OpenSearch: Search (read-only)

4. **DashboardAPIRole**:
   - DynamoDB: GetItem, Query (read-only)
   - S3: GetObject on reports bucket

**User Roles** (Cognito User Pool):

1. **Admin**: Full access to all features, user management
2. **Operations**: View all data, submit copilot queries, make dispatch decisions
3. **MRO_Planner**: View all data, submit copilot queries, schedule maintenance
4. **Read_Only**: View dashboards and reports only

**Authentication**:

- Amazon Cognito User Pool for user authentication
- JWT tokens for API authorization
- MFA required for Admin role
- Session timeout: 8 hours

**API Authorization**:

- API Gateway uses Cognito authorizer
- Lambda functions validate JWT claims
- Fine-grained permissions checked at function level

### Data Segregation

**Multi-Tenancy** (Future):

For production deployment supporting multiple airlines:

1. **Tenant Isolation**:
   - Separate DynamoDB tables per tenant (table-per-tenant model)
   - S3 bucket prefixes per tenant with bucket policies
   - Tenant ID embedded in all data records

2. **Cross-Tenant Prevention**:
   - Lambda functions validate tenant ID from JWT token
   - DynamoDB queries include tenant ID in partition key
   - API Gateway resource policies restrict cross-tenant access

3. **Data Encryption**:
   - Separate KMS keys per tenant
   - Tenant-specific encryption contexts

**Hackathon MVP**:

Single-tenant deployment with architecture designed for future multi-tenancy.

### Responsible AI Considerations

**Transparency**:

- All AI-generated content labeled as "AI-Generated"
- Model versions and training data provenance documented
- Limitations clearly communicated to users

**Fairness**:

- Model performance evaluated across aircraft types
- No bias toward specific manufacturers or operators
- Equal treatment of all UDAN routes regardless of profitability

**Accountability**:

- Complete audit trails for all AI decisions
- Human oversight required for critical decisions
- Clear escalation path for AI errors or concerns

**Safety**:

- Conservative bias: System errs on side of caution
- No autonomous dispatch decisions
- Fail-safe defaults: System unavailability defaults to manual processes

**Privacy**:

- No personal data of passengers or crew
- Anonymized maintenance logs
- Aggregate reporting only (no individual aircraft identification in public reports)

### Hackathon-Safe Data Usage Statement

**Data Ethics Commitment**:

The UDAN MRO Control Tower uses ONLY:

1. **Public Flight Data**: Sourced from open flight tracking APIs (OpenSky Network, ADS-B Exchange)
2. **Open Datasets**: NASA C-MAPSS turbofan degradation dataset, PHM Society challenge datasets
3. **Synthetic Data**: Maintenance logs generated using statistical models based on industry patterns

**Explicitly NOT Used**:

- Proprietary airline operational data
- DGCA-restricted regulatory information
- Confidential maintenance records
- Personal data of any individuals

**Provenance Tracking**:

Every data point in the system includes metadata indicating:
- Source (public API, open dataset, synthetic)
- Collection timestamp
- Data quality indicators

**Hackathon Compliance**:

This data strategy ensures the project:
- Complies with data privacy regulations
- Respects intellectual property rights
- Demonstrates realistic capabilities without requiring restricted data access
- Can be safely demonstrated in public hackathon settings

## Scalability and Future Extensions

### Evolution to Production Deployment

**Phase 1: Hackathon MVP** (Current):
- Single airline, 10 aircraft
- Synthetic and open data only
- Manual data ingestion
- Basic dashboard

**Phase 2: Pilot Deployment** (3-6 months):
- Single airline, 50 aircraft
- Integration with airline's maintenance management system
- Real operational data (with permissions)
- Enhanced predictive models with airline-specific training
- Mobile app for field technicians

**Phase 3: Multi-Airline Platform** (6-12 months):
- Multiple airlines, 200+ aircraft
- Full multi-tenancy with data isolation
- Automated spares procurement integration
- Advanced analytics and reporting
- API marketplace for third-party integrations

**Phase 4: Ecosystem Platform** (12+ months):
- Integration with MRO service providers
- Spares suppliers marketplace
- Regulatory reporting automation
- Industry-wide benchmarking (anonymized)
- Predictive models for new aircraft types

### Scalability Enhancements

**Compute Scaling**:

- Lambda concurrency limits increased from 10 to 1000
- SageMaker endpoints with auto-scaling (1-20 instances)
- Step Functions execution limits increased
- API Gateway throttling adjusted for production load

**Data Scaling**:

- DynamoDB on-demand pricing for unpredictable workloads
- S3 Intelligent-Tiering for cost optimization
- OpenSearch cluster scaling (2-10 nodes)
- ElastiCache cluster mode for distributed caching

**Geographic Distribution**:

- Multi-region deployment for disaster recovery
- CloudFront edge locations for global dashboard access
- DynamoDB Global Tables for cross-region replication
- Regional SageMaker endpoints for low-latency inference

### Integration with DGCA-Compliant Systems (Conceptual)

**Regulatory Reporting**:

Future integration could support automated generation of DGCA-required reports:

- Maintenance event reporting
- Airworthiness directive compliance tracking
- Continuing airworthiness monitoring

**Compliance Checks**:

System could validate maintenance decisions against regulatory requirements:

- Minimum equipment list (MEL) compliance
- Maintenance interval adherence
- Certification requirements

**Audit Support**:

Complete audit trails support DGCA inspections:

- Immutable logs of all maintenance decisions
- Traceability of data sources and reasoning
- Exportable reports in regulatory formats

**Important Note**: Any integration with DGCA systems would require:
- Formal approval and certification processes
- Security audits and compliance verification
- Data sharing agreements
- Adherence to DGCA IT security standards

This design provides the architectural foundation for such integration while maintaining clear separation between advisory functions and regulatory compliance.


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified the following testable properties. During reflection, I consolidated related properties to eliminate redundancy:

**Consolidations Made**:
- Properties 2.5, 3.5, and 12.3 (synthetic data labeling) → Combined into single comprehensive property
- Properties 4.4 and 11.1 (risk score explanations) → Combined as they test the same explainability requirement
- Properties 1.4 and 3.4 (metadata storage) → Combined into general metadata completeness property
- Properties 7.3 and 7.4 (dispatch recommendation completeness) → Combined into single property covering both cases

### Core System Properties

**Property 1: Flight Data Update Triggers Risk Re-evaluation**

*For any* flight delay event received by the system, processing the delay SHALL result in both an updated operational status in the data store AND a risk re-evaluation event being triggered.

**Validates: Requirements 1.2**

**Property 2: Aircraft Type Validation**

*For any* aircraft type string submitted to the Flight_Data_Ingestion, the system SHALL accept it if and only if it matches a supported aircraft configuration in the system's aircraft registry.

**Validates: Requirements 1.3**

**Property 3: Stored Data Includes Required Metadata**

*For any* data record stored in the system (flight operations, maintenance logs, aircraft health), retrieving that record SHALL return data that includes timestamp, provenance source, and data type metadata fields.

**Validates: Requirements 1.4, 3.4**

**Property 4: Ingestion Failure Retry Logic**

*For any* data ingestion operation that fails, the system SHALL attempt exactly 3 retries with exponentially increasing delays, AND SHALL log each failure attempt with error details.

**Validates: Requirements 1.5**

**Property 5: Sensor Reading Component Mapping**

*For any* valid sensor reading from aircraft health data, the system SHALL map it to exactly one aircraft component identifier from the component registry.

**Validates: Requirements 2.1**

**Property 6: Health Data Update Triggers Model Re-evaluation**

*For any* aircraft health data update, the system SHALL trigger either a model re-training workflow OR a prediction re-evaluation, depending on the magnitude of the update.

**Validates: Requirements 2.3**

**Property 7: Health Data Validation Flags Missing Parameters**

*For any* aircraft health data record, if any critical parameter from the required parameter list is missing, the system SHALL flag the record as incomplete and SHALL NOT use it for risk scoring.

**Validates: Requirements 2.4**

**Property 8: Synthetic Data Labeling**

*For any* data record or system output that includes synthetic data, the output SHALL contain an explicit label indicating "synthetic" or "simulated" for that data element.

**Validates: Requirements 2.5, 3.5, 12.3**

**Property 9: Maintenance Log Parsing Round-Trip**

*For any* valid ATA-formatted maintenance log, parsing the log into the internal representation and then serializing it back to ATA format SHALL produce a semantically equivalent log (preserving all key maintenance events and metadata).

**Validates: Requirements 3.1**

**Property 10: Maintenance Event Extraction Completeness**

*For any* maintenance log containing component replacements, inspections, or defect reports, the extraction process SHALL identify and extract ALL such events, with no events omitted.

**Validates: Requirements 3.2**

**Property 11: Parsing Error Messages Are Descriptive**

*For any* malformed maintenance log that fails parsing, the error message SHALL include the line number or record identifier of the problematic entry AND a description of the validation failure.

**Validates: Requirements 3.3**

**Property 12: Risk Score Range and Severity Mapping**

*For any* aircraft and input data, the calculated risk score SHALL be in the range [0, 100], AND SHALL map to severity levels as: [0-33] → Low, [34-66] → Medium, [67-100] → High.

**Validates: Requirements 4.2**

**Property 13: High Risk Score Generates Alert**

*For any* risk score calculation that results in a score ≥ 67, the system SHALL generate an alert record that includes the aircraft identifier, risk score, and at least one recommended action.

**Validates: Requirements 4.3**

**Property 14: Risk Score Includes Explainable Factors**

*For any* risk score generated, the output SHALL include a ranked list of contributing factors with relative importance weights that sum to 1.0.

**Validates: Requirements 4.4, 11.1**

**Property 15: Risk Score Considers Required Inputs**

*For any* two aircraft with identical maintenance history and operational patterns but different utilization levels, the risk scores SHALL differ, demonstrating that utilization is considered in the calculation.

**Validates: Requirements 4.5**

**Property 16: Predictive Model Generates Required Outputs**

*For any* aircraft with sufficient health data, the predictive maintenance model SHALL generate outputs that include failure probability, confidence interval bounds, and remaining useful life estimate.

**Validates: Requirements 5.1, 5.2**

**Property 17: Low Confidence Predictions Are Flagged**

*For any* prediction with confidence < 0.70, the output SHALL include a "low-confidence" flag or warning indicator.

**Validates: Requirements 5.3**

**Property 18: Ambiguous Queries Trigger Clarification**

*For any* natural language query to the AI_Copilot that contains ambiguous references (e.g., "the aircraft" when multiple aircraft are in context), the system SHALL respond with a clarifying question before providing a substantive answer.

**Validates: Requirements 6.3**

**Property 19: Copilot Responses Include Citations**

*For any* response generated by the AI_Copilot, the response SHALL include at least one data source citation AND a confidence level indicator.

**Validates: Requirements 6.4**

**Property 20: Dispatch Decisions Evaluate Required Factors**

*For any* dispatch decision request, the decision process SHALL retrieve and evaluate both aircraft airworthiness status AND current maintenance risk score before generating a recommendation.

**Validates: Requirements 7.1**

**Property 21: Dispatch Recommendations Consider Destination Context**

*For any* two dispatch decision requests for the same aircraft to different destinations with different MRO capabilities, the recommendations SHALL differ if the risk score is in the medium range [34-66], demonstrating that destination context influences recommendations.

**Validates: Requirements 7.2**

**Property 22: Dispatch Recommendations Include Required Information**

*For any* dispatch recommendation, the output SHALL include: (1) the recommendation (dispatch/no-dispatch/caution), (2) reasoning, (3) risk score, AND (4) if recommending dispatch, alternate airports with MRO support, OR if recommending no-dispatch, nearest maintenance facility and estimated TAT.

**Validates: Requirements 7.3, 7.4**

**Property 23: Dispatch Recommendations Include Advisory Statement**

*For any* dispatch recommendation generated, the output SHALL contain a statement indicating the recommendation is advisory and requires human validation.

**Validates: Requirements 7.5**

**Property 24: Predictive Forecasts Trigger Spare Part Identification**

*For any* predictive maintenance forecast indicating component replacement within 30 days, the Spares_Recommender SHALL identify at least one required spare part for that component.

**Validates: Requirements 8.1**

**Property 25: Spares Positioning Considers Predicted Demand**

*For any* two airports with different predicted maintenance demand for the same component, the spares positioning recommendation SHALL allocate more inventory to the higher-demand airport.

**Validates: Requirements 8.2**

**Property 26: Insufficient Inventory Generates Procurement Alert**

*For any* scenario where predicted demand exceeds current inventory plus in-transit spares, the system SHALL generate a procurement alert that includes the part identifier, shortage quantity, and recommended lead time.

**Validates: Requirements 8.3**

**Property 27: Spares Prioritization Favors High-Risk Components**

*For any* two components with different risk scores, when spares budget is constrained, the positioning recommendation SHALL prioritize the higher-risk component.

**Validates: Requirements 8.4**

**Property 28: Spares Decisions Include Cost-Benefit Analysis**

*For any* spares positioning recommendation, the output SHALL include a cost-benefit analysis with at least cost estimate and expected benefit metric (e.g., AOG risk reduction).

**Validates: Requirements 8.5**

**Property 29: Report Export Format Validation**

*For any* report generation request specifying PDF or CSV format, the generated file SHALL be parseable by standard PDF or CSV parsers respectively, with no corruption.

**Validates: Requirements 9.4**

**Property 30: Audit Logs Include Required Fields**

*For any* system recommendation logged to the audit trail, the log entry SHALL include timestamp, input data snapshot, model version, and reasoning chain.

**Validates: Requirements 10.1**

**Property 31: Audit Logs Are Searchable**

*For any* audit log query specifying aircraft registration, date range, or recommendation type, the system SHALL return all matching log entries that satisfy ALL specified criteria.

**Validates: Requirements 10.3**

**Property 32: Audit Logs Are Immutable**

*For any* audit log entry, attempts to modify or delete the entry SHALL fail with an access denied error, regardless of user role.

**Validates: Requirements 10.5**

**Property 33: Predictive Model Explanations Include Feature Importance**

*For any* prediction made by the Predictive_Maintenance_Model, the explanation SHALL include a ranked list of input features with importance scores indicating which features most influenced the prediction.

**Validates: Requirements 11.2**

**Property 34: Multi-Tenant Data Isolation**

*For any* two different airline tenants, a query executed in tenant A's context SHALL return only data belonging to tenant A, with zero data leakage from tenant B.

**Validates: Requirements 13.1**

**Property 35: Role-Based Access Control Enforcement**

*For any* user with role R attempting to access resource X, access SHALL be granted if and only if role R has the required permission for resource X in the access control policy.

**Validates: Requirements 14.2**

**Property 36: Access Attempts Are Logged**

*For any* authentication or authorization attempt (successful or failed), the system SHALL create a log entry that includes user identifier, timestamp, resource accessed, and outcome.

**Validates: Requirements 14.4**

### Edge Cases and Examples

The following are specific examples or edge cases that should be tested with unit tests rather than property-based tests:

**Example 1: Aircraft Health Data Storage with Versioning**

Verify that storing aircraft health data to S3 results in a versioned object with version metadata retrievable via S3 API.

**Validates: Requirements 2.2**

**Example 2: Copilot Query Type Coverage**

Verify that the AI_Copilot can successfully respond to at least one example query from each category: aircraft status, risk factors, maintenance history, and spares availability.

**Validates: Requirements 6.2**

**Example 3: Dashboard Fleet Overview Display**

Verify that accessing the dashboard API endpoint returns risk scores for all aircraft currently in the fleet database.

**Validates: Requirements 9.1**

**Example 4: Dashboard Route Reliability Metrics**

Verify that the dashboard API returns UDAN route-level metrics including on-time performance percentage and maintenance-related delay count.

**Validates: Requirements 9.3**

**Example 5: Detailed Technical Explanations Available**

Verify that requesting a detailed technical explanation for a risk score returns additional technical information beyond the standard user-facing explanation.

**Validates: Requirements 11.5**

**Example 6: Data Catalog Exists and Is Complete**

Verify that the data catalog endpoint returns a list of all data sources with provenance and usage permission information for each source.

**Validates: Requirements 12.1**

**Example 7: Data Ethics Statement Is Public**

Verify that the public-facing data ethics statement is accessible without authentication and contains required sections on data sources and usage principles.

**Validates: Requirements 12.5**

**Example 8: Role Definitions Are Complete**

Verify that the system defines exactly four roles (Admin, Operations, MRO_Planner, Read_Only) with documented permissions for each.

**Validates: Requirements 14.1**

## Error Handling

### Error Categories

**1. Data Ingestion Errors**:
- **Invalid Data Format**: Return HTTP 400 with detailed validation error message
- **API Unavailability**: Retry with exponential backoff, log failure after 3 attempts
- **Schema Validation Failure**: Log to dead-letter queue, alert operations team

**2. Model Inference Errors**:
- **Insufficient Data**: Return error indicating minimum data requirements not met
- **Model Endpoint Unavailable**: Fallback to cached predictions if available, otherwise return error
- **Prediction Timeout**: Return partial results with low-confidence flag

**3. AI Copilot Errors**:
- **Ambiguous Query**: Request clarification from user
- **Tool Invocation Failure**: Inform user of temporary unavailability, suggest retry
- **Context Length Exceeded**: Summarize conversation history, continue with summarized context

**4. Authorization Errors**:
- **Unauthenticated Request**: Return HTTP 401, redirect to login
- **Insufficient Permissions**: Return HTTP 403 with message indicating required role
- **Expired Token**: Return HTTP 401, trigger token refresh flow

**5. System Errors**:
- **Database Unavailability**: Return HTTP 503, trigger CloudWatch alarm
- **Lambda Timeout**: Log partial execution state, return HTTP 504
- **Rate Limit Exceeded**: Return HTTP 429 with retry-after header

### Error Response Format

All API errors follow a consistent JSON structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error description",
    "details": {
      "field": "specific field that caused error",
      "constraint": "constraint that was violated"
    },
    "request_id": "unique request identifier for support",
    "timestamp": "ISO 8601 timestamp"
  }
}
```

### Graceful Degradation

When components fail, the system degrades gracefully:

- **Predictive Model Unavailable**: Risk scoring continues using historical patterns only
- **AI Copilot Unavailable**: Users can still access dashboard and manual queries
- **Real-Time Updates Unavailable**: Dashboard falls back to manual refresh
- **Knowledge Base Unavailable**: Copilot responds with cached knowledge only

### Monitoring and Alerting

CloudWatch alarms trigger for:
- Error rate > 5% for any Lambda function
- SageMaker endpoint latency > 10 seconds
- DynamoDB throttling events
- S3 PUT failure rate > 1%
- API Gateway 5xx error rate > 2%

Alerts are sent to SNS topic subscribed by operations team.

## Testing Strategy

### Dual Testing Approach

The system employs both unit testing and property-based testing for comprehensive coverage:

**Unit Tests**: Focus on specific examples, edge cases, and integration points
- Specific data format examples (valid and invalid)
- Edge cases (empty inputs, boundary values, null handling)
- Error conditions and exception handling
- Integration between components
- Examples listed in "Edge Cases and Examples" section above

**Property-Based Tests**: Verify universal properties across randomized inputs
- All 36 properties listed in "Correctness Properties" section
- Each property test runs minimum 100 iterations with randomized inputs
- Properties validate behavior holds for all valid inputs, not just examples

### Property-Based Testing Configuration

**Framework**: Use appropriate PBT library for implementation language:
- Python: Hypothesis
- TypeScript/JavaScript: fast-check
- Java: jqwik or QuickCheck for Java

**Test Configuration**:
- Minimum 100 iterations per property test
- Seed-based reproducibility for failed tests
- Shrinking enabled to find minimal failing examples

**Test Tagging**:

Each property-based test MUST include a comment tag referencing the design property:

```python
# Feature: udan-mro-control-tower, Property 1: Flight Data Update Triggers Risk Re-evaluation
def test_flight_delay_triggers_risk_reevaluation(flight_delay_event):
    # Test implementation
    pass
```

**Coverage Goals**:
- Unit test coverage: 80%+ for all Lambda functions
- Property test coverage: 100% of properties in design document
- Integration test coverage: All critical user workflows

### Test Data Strategy

**Generators**:

Property-based tests use generators to create randomized test data:

- **Aircraft Registration Generator**: Valid format (VT-XXX)
- **Flight Data Generator**: Valid schedules with realistic delays
- **Maintenance Log Generator**: ATA-compliant log structures
- **Sensor Reading Generator**: Realistic sensor value ranges
- **Risk Score Generator**: Values in [0, 100] range

**Constraints**:

Generators respect domain constraints:
- Timestamps are chronologically ordered
- Aircraft types are from supported list
- ATA chapters are valid (00-99)
- Sensor readings are within physical limits

### Integration Testing

**End-to-End Workflows**:

1. **Data Ingestion to Risk Scoring**: Ingest flight data → Verify risk score updated
2. **Predictive Maintenance to Spares Recommendation**: Health data → Prediction → Spares alert
3. **Copilot Query to Response**: User query → Tool invocation → Response with citations
4. **Dispatch Decision Workflow**: Request → Risk evaluation → Recommendation with reasoning

**AWS Service Integration**:

- Mock AWS services in unit tests using moto or localstack
- Use actual AWS services in integration tests (separate test account)
- Clean up test resources after each test run

### Performance Testing

While not part of unit/property tests, performance requirements are validated through:

- Load testing with Locust or Artillery
- Latency monitoring in staging environment
- SageMaker endpoint performance benchmarks
- API Gateway throttling tests

Performance targets from requirements:
- Risk score calculation: < 5 seconds (95th percentile)
- Copilot response: < 3 seconds (90th percentile)
- Dashboard load: < 2 seconds (95th percentile)

### Continuous Testing

**CI/CD Pipeline**:

1. **Pre-Commit**: Run unit tests locally
2. **Pull Request**: Run all unit and property tests
3. **Merge to Main**: Run integration tests
4. **Staging Deployment**: Run end-to-end tests
5. **Production Deployment**: Run smoke tests

**Test Automation**:

- GitHub Actions or AWS CodePipeline for CI/CD
- Automated test execution on every commit
- Test results published to dashboard
- Failed tests block deployment

## Conclusion

This design document provides a comprehensive architecture for the UDAN MRO Control Tower, an AI-powered maintenance decision-support system built on AWS. The system leverages modern cloud services, machine learning, and agentic AI to address the unique challenges of regional aircraft maintenance in India's UDAN scheme.

Key architectural highlights:

- **Serverless-first design** using Lambda, Step Functions, and managed services for scalability
- **Agentic AI copilot** powered by Amazon Bedrock for natural language maintenance queries
- **Predictive maintenance** using SageMaker-hosted ML models with explainable outputs
- **RAG-enabled knowledge base** for grounding AI responses in maintenance documentation
- **Complete auditability** with immutable decision traces and explainable reasoning
- **Production-ready patterns** including multi-tenancy, RBAC, and graceful degradation

The design balances hackathon feasibility with production-grade architectural principles, providing a clear path from MVP to full-scale deployment. All components are designed with explainability, auditability, and safety as core principles, ensuring the system serves as a trustworthy decision-support tool for critical aviation maintenance operations.
