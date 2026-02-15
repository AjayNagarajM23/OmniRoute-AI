# Design Document: OmniRoute AI

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Presentation Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Web UI     │  │  REST API    │  │  Notification Service│  │
│  │  (React)     │  │  (FastAPI)   │  │     (WebSocket)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Orchestration Layer                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Multi-Agent Coordinator (LangGraph)              │  │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │  │
│  │  │  Tariff    │  │  Supplier  │  │  Optimization    │   │  │
│  │  │  Agent     │  │  Intel     │  │  Trigger Agent   │   │  │
│  │  │            │  │  Agent     │  │                  │   │  │
│  │  └────────────┘  └────────────┘  └──────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                       Intelligence Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Gemini API │  │  Forecasting │  │   Sentiment Engine   │  │
│  │  (Document   │  │   Engine     │  │   (NLP Pipeline)     │  │
│  │  Processing) │  │ (XGBoost)    │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Optimization Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Gurobi     │  │ OR-Tools TSP │  │  Warehouse Slotting  │  │
│  │   Sourcing   │  │     VRP      │  │    Optimizer         │  │
│  │     MIP      │  │   Solver     │  │   (OR-Tools CP)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                         Data Layer                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  PostgreSQL  │  │    Redis     │  │   Vector DB          │  │
│  │  (Relational)│  │   (Cache)    │  │  (Supplier Docs)     │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Descriptions

#### 1.2.1 Presentation Layer
- **Web UI**: React-based dashboard with real-time updates
- **REST API**: FastAPI endpoints for CRUD operations and optimization triggers
- **Notification Service**: WebSocket server for real-time alerts

#### 1.2.2 Orchestration Layer
- **Multi-Agent Coordinator**: LangGraph state machine managing agent lifecycles
- **Agents**:
  - Tariff Agent: Monitors trade policy sources
  - Supplier Intel Agent: Scrapes reviews and news
  - Optimization Trigger Agent: Detects when re-optimization is needed

#### 1.2.3 Intelligence Layer
- **Gemini API**: Document understanding for tariff PDFs and news articles
- **Forecasting Engine**: Time-series models (XGBoost/Prophet)
- **Sentiment Engine**: NLP pipeline for review analysis

#### 1.2.4 Optimization Layer
- **Gurobi MIP**: Sourcing optimization solver
- **OR-Tools TSP/VRP**: Route optimization
- **Warehouse Slotting**: Constraint programming for slot assignment

#### 1.2.5 Data Layer
- **PostgreSQL**: Transactional data (suppliers, orders, tariffs)
- **Redis**: Caching and agent state
- **Vector DB**: Embeddings for supplier document search

## 2. Detailed Component Design

### 2.1 Multi-Agent System (LangGraph)

#### 2.1.1 Agent Graph Structure

```python
# State definition
class SupplyChainState(TypedDict):
    tariff_updates: List[TariffChange]
    supplier_scores: Dict[str, float]
    demand_forecast: Dict[str, float]
    optimization_trigger: bool
    sourcing_plan: Optional[SourcingPlan]
    
# Agent nodes
graph = StateGraph(SupplyChainState)
graph.add_node("tariff_monitor", tariff_agent)
graph.add_node("supplier_intel", supplier_agent)
graph.add_node("demand_forecast", forecast_agent)
graph.add_node("sourcing_optimizer", sourcing_agent)
graph.add_node("warehouse_optimizer", warehouse_agent)
graph.add_node("route_optimizer", route_agent)

# Conditional edges
graph.add_conditional_edges(
    "tariff_monitor",
    should_reoptimize,
    {True: "sourcing_optimizer", False: END}
)
```

#### 2.1.2 Agent Specifications

**Tariff Monitoring Agent**
- **Input**: List of monitored countries and product categories
- **Process**:
  1. Scrape government trade portals (e.g., USTR, WTO)
  2. Parse PDFs using Gemini API
  3. Extract tariff rates and effective dates
  4. Compare with historical data
- **Output**: List of TariffChange objects
- **Trigger**: Scheduled (every 6 hours) or on-demand

**Supplier Intelligence Agent**
- **Input**: Supplier IDs from database
- **Process**:
  1. Scrape B2B platforms (Alibaba, ThomasNet)
  2. Collect reviews, ratings, and news mentions
  3. Perform sentiment analysis using Gemini
  4. Calculate trust score using weighted formula
- **Output**: Updated supplier_scores dictionary
- **Trigger**: Daily batch + event-driven (new review alert)

**Optimization Trigger Agent**
- **Input**: Current state (tariffs, supplier scores, demand)
- **Process**:
  1. Calculate delta from last optimization
  2. Check thresholds (tariff >5%, supplier score >15%, demand >10%)
  3. Set optimization_trigger flag
- **Output**: Boolean trigger
- **Trigger**: After tariff/supplier updates

### 2.2 Sourcing Optimization (Gurobi MIP)

#### 2.2.1 Mathematical Model

**Decision Variables:**
- `x[s,p]`: Quantity of product `p` sourced from supplier `s`
- `y[s]`: Binary variable (1 if supplier `s` is selected, 0 otherwise)

**Objective Function:**
```
Minimize: Σ(s,p) [landed_cost[s,p] * x[s,p]] + Σ(s) [fixed_cost[s] * y[s]]

where landed_cost[s,p] = base_price[s,p] + tariff[s,p] + shipping[s,p] + risk_premium[s]
```

**Constraints:**
1. Demand satisfaction: `Σ(s) x[s,p] >= demand[p]` for all products `p`
2. Supplier capacity: `Σ(p) x[s,p] <= capacity[s] * y[s]` for all suppliers `s`
3. Minimum order: `x[s,p] >= MOQ[s,p] * y[s]` or `x[s,p] = 0`
4. Quality threshold: `quality_score[s] >= min_quality` if `y[s] = 1`
5. Geographic diversification: `Σ(s in region_r) y[s] <= max_suppliers_per_region`

#### 2.2.2 Implementation

```python
import gurobipy as gp
from gurobipy import GRB

def optimize_sourcing(suppliers, products, demand, tariffs):
    model = gp.Model("sourcing")
    
    # Decision variables
    x = model.addVars(suppliers, products, name="quantity")
    y = model.addVars(suppliers, vtype=GRB.BINARY, name="selected")
    
    # Objective
    landed_cost = {
        (s, p): calc_landed_cost(s, p, tariffs)
        for s in suppliers for p in products
    }
    model.setObjective(
        gp.quicksum(landed_cost[s,p] * x[s,p] for s in suppliers for p in products),
        GRB.MINIMIZE
    )
    
    # Constraints
    # (Add constraints as per model above)
    
    model.optimize()
    return extract_solution(x, y)
```

### 2.3 Warehouse Slotting (OR-Tools CP-SAT)

#### 2.3.1 Optimization Model

**Decision Variables:**
- `slot[i]`: Slot assigned to SKU `i`

**Objective:**
```
Minimize: Σ(i) [pick_frequency[i] * distance_to_packing[slot[i]]]
```

**Constraints:**
1. One SKU per slot: `AllDifferent(slot[i] for all i)`
2. Size compatibility: `sku_size[i] <= slot_capacity[slot[i]]`
3. Complementary grouping: `distance(slot[i], slot[j]) <= max_dist` if `complementary(i, j)`

#### 2.3.2 Implementation

```python
from ortools.sat.python import cp_model

def optimize_slotting(skus, slots, warehouse_layout):
    model = cp_model.CpModel()
    
    # Variables
    slot_vars = {
        sku: model.NewIntVar(0, len(slots)-1, f"slot_{sku}")
        for sku in skus
    }
    
    # Constraints
    model.AddAllDifferent(slot_vars.values())
    
    # Objective (linearized)
    cost_vars = []
    for sku in skus:
        cost_var = model.NewIntVar(0, 10000, f"cost_{sku}")
        # Add distance calculation constraints
        cost_vars.append(cost_var * pick_frequency[sku])
    
    model.Minimize(sum(cost_vars))
    
    solver = cp_model.CpSolver()
    status = solver.Solve(model)
    return extract_slotting(slot_vars, solver)
```

### 2.4 Route Optimization (OR-Tools TSP/VRP)

#### 2.4.1 VRP Model

**Problem Type:** Capacitated VRP with Time Windows (CVRPTW)

**Components:**
- Depot location
- Customer locations with demand and time windows
- Fleet of vehicles with capacity constraints

#### 2.4.2 Implementation

```python
from ortools.constraint_solver import routing_enums_pb2
from ortools.constraint_solver import pywrapcp

def optimize_routes(depot, customers, vehicles, traffic_data):
    manager = pywrapcp.RoutingIndexManager(
        len(customers) + 1,  # +1 for depot
        len(vehicles),
        0  # depot index
    )
    routing = pywrapcp.RoutingModel(manager)
    
    # Distance callback with real-time traffic
    def distance_callback(from_idx, to_idx):
        from_node = manager.IndexToNode(from_idx)
        to_node = manager.IndexToNode(to_idx)
        return get_distance_with_traffic(from_node, to_node, traffic_data)
    
    transit_callback_index = routing.RegisterTransitCallback(distance_callback)
    routing.SetArcCostEvaluatorOfAllVehicles(transit_callback_index)
    
    # Capacity constraints
    def demand_callback(from_idx):
        node = manager.IndexToNode(from_idx)
        return customers[node].demand if node > 0 else 0
    
    demand_callback_index = routing.RegisterUnaryTransitCallback(demand_callback)
    routing.AddDimensionWithVehicleCapacity(
        demand_callback_index,
        0,  # null capacity slack
        [v.capacity for v in vehicles],
        True,  # start cumul to zero
        'Capacity'
    )
    
    # Time window constraints
    routing.AddDimension(
        transit_callback_index,
        30,  # allow 30 min waiting time
        180,  # max 3 hours per route
        False,
        'Time'
    )
    
    search_parameters = pywrapcp.DefaultRoutingSearchParameters()
    search_parameters.first_solution_strategy = (
        routing_enums_pb2.FirstSolutionStrategy.PATH_CHEAPEST_ARC
    )
    
    solution = routing.SolveWithParameters(search_parameters)
    return extract_routes(manager, routing, solution)
```

### 2.5 Demand Forecasting

#### 2.5.1 Model Architecture

**Approach:** Ensemble of XGBoost and Prophet

**Features:**
- Historical sales (lag features: 7, 14, 30, 90 days)
- Calendar features (day of week, month, holidays)
- Promotional flags
- External factors (weather, events)
- Tariff change indicators

**Implementation:**
```python
import xgboost as xgb
from prophet import Prophet

def forecast_demand(sku, historical_data, horizon=90):
    # XGBoost model
    X_train, y_train = prepare_features(historical_data)
    xgb_model = xgb.XGBRegressor(
        objective='reg:squarederror',
        n_estimators=100,
        max_depth=6
    )
    xgb_model.fit(X_train, y_train)
    xgb_forecast = xgb_model.predict(X_future)
    
    # Prophet model
    prophet_df = historical_data[['ds', 'y']]
    prophet_model = Prophet(
        yearly_seasonality=True,
        weekly_seasonality=True
    )
    prophet_model.fit(prophet_df)
    prophet_forecast = prophet_model.predict(future_dates)
    
    # Ensemble (weighted average)
    final_forecast = 0.6 * xgb_forecast + 0.4 * prophet_forecast['yhat']
    
    return final_forecast
```

## 3. Data Models

### 3.1 Database Schema (PostgreSQL)

```sql
-- Suppliers
CREATE TABLE suppliers (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    country VARCHAR(2),
    trust_score DECIMAL(5,2),
    capacity INTEGER,
    certifications JSONB,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Products
CREATE TABLE products (
    id UUID PRIMARY KEY,
    sku VARCHAR(50) UNIQUE,
    name VARCHAR(255),
    hs_code VARCHAR(10),
    category VARCHAR(100),
    dimensions JSONB,
    created_at TIMESTAMP
);

-- Tariffs
CREATE TABLE tariffs (
    id UUID PRIMARY KEY,
    country_from VARCHAR(2),
    country_to VARCHAR(2),
    hs_code VARCHAR(10),
    rate DECIMAL(5,2),
    effective_date DATE,
    source_url TEXT,
    created_at TIMESTAMP
);

-- Supplier Products (pricing)
CREATE TABLE supplier_products (
    supplier_id UUID REFERENCES suppliers(id),
    product_id UUID REFERENCES products(id),
    base_price DECIMAL(10,2),
    moq INTEGER,
    lead_time_days INTEGER,
    PRIMARY KEY (supplier_id, product_id)
);

-- Demand Forecasts
CREATE TABLE demand_forecasts (
    id UUID PRIMARY KEY,
    product_id UUID REFERENCES products(id),
    forecast_date DATE,
    quantity DECIMAL(10,2),
    confidence_lower DECIMAL(10,2),
    confidence_upper DECIMAL(10,2),
    model_version VARCHAR(50),
    created_at TIMESTAMP
);

-- Sourcing Plans
CREATE TABLE sourcing_plans (
    id UUID PRIMARY KEY,
    plan_date DATE,
    status VARCHAR(20),
    total_cost DECIMAL(12,2),
    optimization_time_ms INTEGER,
    created_at TIMESTAMP
);

CREATE TABLE sourcing_plan_items (
    plan_id UUID REFERENCES sourcing_plans(id),
    supplier_id UUID REFERENCES suppliers(id),
    product_id UUID REFERENCES products(id),
    quantity INTEGER,
    landed_cost DECIMAL(10,2),
    PRIMARY KEY (plan_id, supplier_id, product_id)
);

-- Warehouse Slots
CREATE TABLE warehouse_slots (
    id UUID PRIMARY KEY,
    warehouse_id UUID,
    slot_code VARCHAR(20),
    coordinates JSONB,  -- {x, y, z}
    capacity_cubic_meters DECIMAL(6,2),
    current_sku VARCHAR(50),
    updated_at TIMESTAMP
);

-- Delivery Routes
CREATE TABLE delivery_routes (
    id UUID PRIMARY KEY,
    route_date DATE,
    vehicle_id UUID,
    total_distance_km DECIMAL(8,2),
    total_time_minutes INTEGER,
    status VARCHAR(20),
    created_at TIMESTAMP
);

CREATE TABLE route_stops (
    route_id UUID REFERENCES delivery_routes(id),
    stop_sequence INTEGER,
    customer_id UUID,
    address JSONB,
    time_window_start TIME,
    time_window_end TIME,
    estimated_arrival TIME,
    PRIMARY KEY (route_id, stop_sequence)
);
```

### 3.2 Agent State Schema (Redis)

```json
{
  "agent:tariff_monitor": {
    "status": "running",
    "last_run": "2026-02-14T06:00:00Z",
    "next_run": "2026-02-14T12:00:00Z",
    "monitored_countries": ["US", "CN", "EU"],
    "last_changes": [
      {
        "country": "US",
        "hs_code": "8471.30",
        "old_rate": 0.0,
        "new_rate": 25.0,
        "effective_date": "2026-03-01"
      }
    ]
  },
  "agent:supplier_intel": {
    "status": "idle",
    "last_run": "2026-02-14T08:00:00Z",
    "suppliers_processed": 150,
    "alerts": [
      {
        "supplier_id": "uuid-123",
        "trust_score_change": -18.5,
        "reason": "negative_reviews"
      }
    ]
  }
}
```

## 4. API Design

### 4.1 REST Endpoints

```
# Suppliers
GET    /api/v1/suppliers
GET    /api/v1/suppliers/{id}
POST   /api/v1/suppliers
PUT    /api/v1/suppliers/{id}
GET    /api/v1/suppliers/{id}/trust-score-history

# Tariffs
GET    /api/v1/tariffs?country_from={}&country_to={}&hs_code={}
GET    /api/v1/tariffs/changes?since={}

# Forecasting
POST   /api/v1/forecasts/demand
GET    /api/v1/forecasts/{product_id}?horizon=90

# Sourcing Optimization
POST   /api/v1/sourcing/optimize
GET    /api/v1/sourcing/plans/{id}
POST   /api/v1/sourcing/scenarios  # What-if analysis

# Warehouse
POST   /api/v1/warehouse/optimize-slotting
GET    /api/v1/warehouse/slots?warehouse_id={}

# Routes
POST   /api/v1/routes/optimize
GET    /api/v1/routes/{id}
GET    /api/v1/routes/{id}/map

# Agents
GET    /api/v1/agents/status
POST   /api/v1/agents/{agent_name}/trigger
```

### 4.2 WebSocket Events

```
# Client subscribes
-> {"type": "subscribe", "channels": ["tariff_alerts", "optimization_complete"]}

# Server pushes
<- {"type": "tariff_alert", "data": {...}}
<- {"type": "optimization_complete", "plan_id": "uuid-456"}
<- {"type": "supplier_risk", "supplier_id": "uuid-789", "score": 45.2}
```

## 5. Deployment Architecture

### 5.1 Infrastructure (AWS)

```
┌─────────────────────────────────────────────────────────┐
│                     CloudFront (CDN)                     │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                  Application Load Balancer               │
└─────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
┌───────────────┐                  ┌────────────────┐
│   ECS Fargate │                  │  ECS Fargate   │
│   (API)       │                  │  (Agents)      │
│   Auto-scaling│                  │  Spot instances│
└───────────────┘                  └────────────────┘
        │                                   │
        └─────────────────┬─────────────────┘
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
┌───────────────┐                  ┌────────────────┐
│   RDS         │                  │  ElastiCache   │
│   PostgreSQL  │                  │  Redis         │
│   Multi-AZ    │                  │                │
└───────────────┘                  └────────────────┘
```

### 5.2 Container Architecture

**API Service (Dockerfile)**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Agent Service (Dockerfile)**
```dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y gurobi-optimizer
WORKDIR /app
COPY requirements-agents.txt .
RUN pip install --no-cache-dir -r requirements-agents.txt
COPY . .
CMD ["python", "agent_runner.py"]
```

## 6. Security Design

### 6.1 Authentication & Authorization

- **API Authentication**: JWT tokens with 1-hour expiration
- **RBAC Roles**:
  - `admin`: Full access
  - `planner`: Read + trigger optimizations
  - `viewer`: Read-only dashboard access
- **Service-to-Service**: AWS IAM roles for ECS tasks

### 6.2 Data Protection

- **Encryption at Rest**: RDS encryption enabled, S3 bucket encryption
- **Encryption in Transit**: TLS 1.3 for all API calls
- **Secrets Management**: AWS Secrets Manager for API keys (Gemini, Gurobi license)

### 6.3 Network Security

- **VPC**: Private subnets for databases and agents
- **Security Groups**: Whitelist only necessary ports
- **WAF**: CloudFront WAF rules to prevent DDoS

## 7. Monitoring & Observability

### 7.1 Metrics (CloudWatch)

- API latency (p50, p95, p99)
- Agent execution time
- Optimization solver time
- Database connection pool usage
- Cache hit rate

### 7.2 Logging

- Structured JSON logs (ECS → CloudWatch Logs)
- Log levels: DEBUG (dev), INFO (prod)
- Correlation IDs for request tracing

### 7.3 Alerting

- PagerDuty integration for critical alerts:
  - Agent failure (3 consecutive failures)
  - Optimization timeout (>10 minutes)
  - API error rate >5%
  - Database CPU >80%

## 8. Testing Strategy

### 8.1 Unit Tests
- Agent logic (mocked external APIs)
- Optimization model correctness (small test cases)
- API endpoint validation

### 8.2 Integration Tests
- End-to-end agent workflows
- Database transactions
- API + optimization pipeline

### 8.3 Performance Tests
- Load testing: 100 concurrent API requests
- Optimization scalability: 1000 suppliers, 10000 SKUs
- Route optimization: 200 stops

## 9. Future Enhancements

### Phase 2 (6-12 months)
- Multi-modal transportation (air, sea, rail)
- Carbon footprint tracking and optimization
- Blockchain integration for supplier verification
- Mobile app for delivery drivers

### Phase 3 (12-18 months)
- Predictive maintenance for warehouse equipment
- Autonomous negotiation agents for supplier contracts
- Integration with IoT sensors for real-time inventory
- Generative AI for supply chain scenario planning
