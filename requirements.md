# Requirements Document: OmniRoute AI

## 1. Functional Requirements

### 1.1 Intelligence Layer (Agentic AI)

#### FR-1.1: Tariff Monitoring Agent
- **FR-1.1.1**: Continuously monitor global trade policy sources for tariff updates
- **FR-1.1.2**: Parse and extract tariff rate changes by country, product category, and HS codes
- **FR-1.1.3**: Generate alerts when tariff changes exceed configurable thresholds (e.g., >5%)
- **FR-1.1.4**: Maintain historical tariff database with version control

#### FR-1.2: Supplier Intelligence Agent
- **FR-1.2.1**: Scrape B2B review platforms (Alibaba, ThomasNet, etc.) for supplier ratings
- **FR-1.2.2**: Perform sentiment analysis on supplier-related news and social media
- **FR-1.2.3**: Calculate dynamic "Supplier Trust Score" (0-100) based on:
  - Review sentiment (40%)
  - Delivery reliability (30%)
  - Quality metrics (20%)
  - Ethical compliance flags (10%)
- **FR-1.2.4**: Flag suppliers with declining trust scores (>15% drop in 30 days)

#### FR-1.3: Multi-Agent Orchestration
- **FR-1.3.1**: Coordinate agent workflows using LangGraph/AutoGen
- **FR-1.3.2**: Enable inter-agent communication for data sharing
- **FR-1.3.3**: Implement agent state persistence and recovery

### 1.2 Tactical Layer (Demand & Sourcing)

#### FR-2.1: Demand Forecasting
- **FR-2.1.1**: Predict raw material demand for 30/60/90-day horizons
- **FR-2.1.2**: Incorporate seasonality, trends, and external factors (holidays, events)
- **FR-2.1.3**: Provide confidence intervals for forecasts
- **FR-2.1.4**: Support SKU-level and category-level predictions

#### FR-2.2: Sourcing Optimization
- **FR-2.2.1**: Calculate "Landed Cost" = Base Price + Tariff + Shipping + Risk Premium
- **FR-2.2.2**: Solve MIP model to select optimal supplier mix with constraints:
  - Minimum order quantities (MOQ)
  - Supplier capacity limits
  - Quality thresholds
  - Geographic diversification requirements
- **FR-2.2.3**: Generate alternative sourcing scenarios (what-if analysis)
- **FR-2.2.4**: Re-optimize automatically when tariffs change >5%

### 1.3 Execution Layer (Warehouse & Logistics)

#### FR-3.1: Warehouse Slotting Optimization
- **FR-3.1.1**: Assign inventory to warehouse slots based on:
  - Forecasted demand velocity
  - Product dimensions and weight
  - Pick frequency
  - Complementary product grouping
- **FR-3.1.2**: Minimize average pick distance
- **FR-3.1.3**: Support multi-warehouse scenarios
- **FR-3.1.4**: Re-slot dynamically when demand patterns shift

#### FR-3.2: Last-Mile Delivery Optimization (TSP/VRP)
- **FR-3.2.1**: Solve TSP for single-vehicle routes
- **FR-3.2.2**: Solve VRP (Vehicle Routing Problem) for multi-vehicle fleets with:
  - Vehicle capacity constraints
  - Time windows for deliveries
  - Driver shift limits
- **FR-3.2.3**: Incorporate real-time traffic data
- **FR-3.2.4**: Minimize total distance and fuel consumption
- **FR-3.2.5**: Provide turn-by-turn route instructions

### 1.4 User Interface & Reporting

#### FR-4.1: Dashboard
- **FR-4.1.1**: Display real-time KPIs:
  - Current tariff exposure by country
  - Top 10 suppliers by risk score
  - Forecasted demand vs. actual
  - Logistics cost per delivery
- **FR-4.1.2**: Interactive map showing supplier locations, warehouses, and delivery routes
- **FR-4.1.3**: Alert panel for critical events

#### FR-4.2: Scenario Planning
- **FR-4.2.1**: Allow users to simulate tariff changes
- **FR-4.2.2**: Compare sourcing strategies side-by-side
- **FR-4.2.3**: Export optimization results to CSV/Excel

#### FR-4.3: Audit Trail
- **FR-4.3.1**: Log all agent decisions with timestamps
- **FR-4.3.2**: Track optimization model inputs and outputs
- **FR-4.3.3**: Maintain compliance records for supplier vetting

## 2. Non-Functional Requirements

### 2.1 Performance
- **NFR-2.1.1**: Tariff monitoring agents check sources every 6 hours
- **NFR-2.1.2**: Sourcing optimization completes within 5 minutes for 100 suppliers
- **NFR-2.1.3**: TSP/VRP solutions for 50 stops complete within 2 minutes
- **NFR-2.1.4**: Dashboard loads within 3 seconds

### 2.2 Scalability
- **NFR-2.2.1**: Support 1,000+ suppliers in sourcing database
- **NFR-2.2.2**: Handle 10,000+ SKUs for demand forecasting
- **NFR-2.2.3**: Optimize routes for fleets up to 100 vehicles

### 2.3 Reliability
- **NFR-2.3.1**: System uptime of 99.5%
- **NFR-2.3.2**: Agent failure recovery within 5 minutes
- **NFR-2.3.3**: Data backup every 24 hours

### 2.4 Security
- **NFR-2.4.1**: Encrypt supplier data at rest and in transit
- **NFR-2.4.2**: Role-based access control (RBAC) for users
- **NFR-2.4.3**: API authentication using OAuth 2.0
- **NFR-2.4.4**: Audit logs retained for 2 years

### 2.5 Usability
- **NFR-2.5.1**: Dashboard accessible on desktop and tablet
- **NFR-2.5.2**: Onboarding tutorial for new users
- **NFR-2.5.3**: Contextual help tooltips

### 2.6 Compliance
- **NFR-2.6.1**: GDPR compliance for supplier data
- **NFR-2.6.2**: SOC 2 Type II certification readiness
- **NFR-2.6.3**: Export control compliance for tariff data

## 3. Data Requirements

### 3.1 Input Data
- **DR-3.1.1**: Tariff schedules (CSV/XML from customs agencies)
- **DR-3.1.2**: Supplier master data (name, location, contact, certifications)
- **DR-3.1.3**: Historical purchase orders and invoices
- **DR-3.1.4**: Warehouse layout (slot coordinates, dimensions)
- **DR-3.1.5**: Delivery addresses with geocodes
- **DR-3.1.6**: Real-time traffic feeds (Google Maps API)

### 3.2 Output Data
- **DR-3.2.1**: Optimized sourcing plans (supplier, quantity, cost)
- **DR-3.2.2**: Warehouse slotting assignments
- **DR-3.2.3**: Delivery route manifests
- **DR-3.2.4**: Risk reports and alerts

## 4. Integration Requirements

### 4.1 External APIs
- **IR-4.1.1**: Gemini API for document understanding and sentiment analysis
- **IR-4.1.2**: Google Maps API for geocoding and traffic data
- **IR-4.1.3**: Web scraping endpoints for B2B platforms (with rate limiting)

### 4.2 Internal Systems
- **IR-4.2.1**: ERP integration for purchase order creation
- **IR-4.2.2**: WMS (Warehouse Management System) for slotting updates
- **IR-4.2.3**: TMS (Transportation Management System) for route execution

## 5. Constraints & Assumptions

### 5.1 Constraints
- **C-5.1.1**: Gurobi license required for MIP optimization (commercial license)
- **C-5.1.2**: Google OR-Tools open-source license (Apache 2.0)
- **C-5.1.3**: Gemini API rate limits (60 requests/minute)

### 5.2 Assumptions
- **A-5.2.1**: Supplier data is available in structured format
- **A-5.2.2**: Tariff sources provide machine-readable data
- **A-5.2.3**: Users have basic understanding of supply chain concepts
- **A-5.2.4**: Internet connectivity available for real-time data feeds

## 6. Success Metrics

- **SM-6.1**: Reduce sourcing costs by 15% within 6 months
- **SM-6.2**: Detect supplier quality issues 30 days earlier than manual review
- **SM-6.3**: Decrease logistics costs by 10% through route optimization
- **SM-6.4**: Achieve 95% forecast accuracy (MAPE) for top 20% SKUs
- **SM-6.5**: Respond to tariff changes within 24 hours with new sourcing plan
