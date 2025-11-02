# Agentic AI Travel Itinerary System – Project Document (Starter Version)

---

## 1. Overall Layer, Architecture, Design Pattern, Kafka Topics, Partitions

### Functional Layers

* **User Layer**
* **Orchestration Layer**
* **Communication Layer**
* **Agent Layer**
* **Integration/External Layer**
* **Data Layer**
* **External System Layer**

### Architecture Pattern

We can use a **Hub-and-Spoke** orchestration with an agent as the central controller. The system follows an **event-driven** model using **Kafka** brokers, producers, subscribers, topics, and consumer groups.
The **Data Layer** can act as a **blackboard** using **Redis**, **Postgres**, and **Vector DBs** for short-term and long-term memory.
Reasoning is **LLM-driven**, and scalability is achieved via **micro-agents**.

### Design Pattern

* **Reasoning Pattern** for main orchestrator
* **Optimizer Agent** can use LLM reasoning
* **Memory**: Short-term and long-term via Redis/Postgres/Vector DB
* **Tool Use**: MCP Server for external API access
* **Reflection**: Pre-planner agent re-evaluates plans based on new data

---

## 2. Kafka Topics and Partitions

| Topic                 | Partition Key |
| --------------------- | ------------- |
| travel.requests       | user_id       |
| agent.tasks           | task_type     |
| agent.results         | agent_name    |
| optimization.requests | request_type  |
| disruption.events     | event_type    |

---

### 2.1 Partition Distribution

(Conceptual: every unique customer request goes to the same partition)

#### **travel.requests (3 partitions)**

* P0: Simple requests (single destination, standard budget)
* P1: Complex requests (multi-city, custom preferences)
* P2: Urgent requests (immediate travel, replanning)

#### **agent.tasks (5 partitions)**

* P0: Weather analysis tasks
* P1: Budget optimization tasks
* P2: Event detection tasks
* P3: Transport routing tasks
* P4: Preference learning tasks

#### **agent.results (5 partitions)**

* P0: Weather agent results
* P1: Budget agent results
* P2: Event agent results
* P3: Transport agent results
* P4: Preference agent results

#### **optimization.requests (2 partitions)**

* P0: Initial itinerary optimization requests
* P1: Re-optimization requests triggered by disruptions (e.g., flight delay, weather)

#### **disruption.events (3 partitions)**

* P0: Flight delay and cancellation alerts
* P1: Severe weather warnings impacting destinations
* P2: Infrastructure disruptions (road closures, strikes, etc.)

---

## 3. Expected Deliverables

### System Architecture Diagram with Data and Control Flow

**Architecture Layers**

1. User Interface – Input destination, dates, budget, preferences
2. Main Orchestrator – LLM-based reasoning and coordination
3. Message Bus – Kafka or RabbitMQ for async task distribution
4. Specialized Agents – Weather, Budget, Event, Transport, Preference
5. Optimizer Agent – Combines results and generates itinerary
6. Replanner Agent – Monitors for dynamic events and triggers re-optimization
7. MCP Server – Handles authentication, rate limiting, caching, and logging
8. Data Layer – Stores orchestration logs, optimization results, and preferences
9. External APIs – Provide weather, travel, map, and event data

### Snapshots

![System Architecture Snapshot 1](https://github.com/sam2881/travel_agent/blob/master/images/snap1.png?raw=true)
![System Architecture Snapshot 2](https://github.com/sam2881/travel_agent/blob/master/images/snap2.png?raw=true)

---

## 4. Part 1: Simple Use Case

### Example: Weather-based Adjustment

**Scenario:** User wants to visit Goa in July → Suggest indoor attractions or alternate months (Dec–Feb)

| Step | Action                                                                                               |
| ---- | ---------------------------------------------------------------------------------------------------- |
| 1    | User submits travel request — “Plan a trip to Goa in July.”                                          |
| 2    | Main Orchestrator receives the request, extracts intent, and publishes async tasks to Message Bus.   |
| 3    | Message Bus distributes tasks to specialized agents (Weather, Budget, Event, Transport, Preference). |
| 4    | Weather Agent queries MCP Server for live and historical weather data.                               |
| 5    | MCP Server authenticates and calls external Weather APIs (OpenWeather, Meteo).                       |
| 6    | APIs return rainfall, humidity, and temperature forecasts for July in Goa.                           |
| 7    | Weather Agent analyzes data, detects poor conditions, and sends warnings to Message Bus.             |
| 8    | Other agents (Budget, Event, Transport, Preference) execute normally and send results.               |
| 9    | Message Bus aggregates all responses and sends to Main Orchestrator.                                 |
| 10   | Main Orchestrator detects unfavorable weather.                                                       |
| 11   | Orchestrator sends context to Optimizer Agent for reasoning.                                         |
| 12   | Optimizer Agent suggests alternatives (indoor activities, better travel months).                     |
| 13   | Optimizer Agent returns recommendations to Orchestrator.                                             |
| 14   | Main Orchestrator compiles final itinerary.                                                          |
| 15   | User Interface presents personalized, weather-aware itinerary.                                       |

---

## 5. Part 2: Advanced / Difficult Use Case

### Scenario: Dynamic Re-planning – Flight delayed by 6 hours

| Step | Action Summary                                                                      |
| ---- | ----------------------------------------------------------------------------------- |
| 1    | User submits travel request (destination, dates, budget, preferences).              |
| 2    | Orchestrator publishes tasks to Message Bus.                                        |
| 3–7  | Specialized agents process Weather, Budget, Event, Transport, and Preference tasks. |
| 8    | Agents send results back to Message Bus.                                            |
| 9    | Message Bus aggregates and returns results to Orchestrator.                         |
| 10   | Orchestrator sends data to Optimizer Agent.                                         |
| 11   | Optimizer returns optimized itinerary.                                              |
| 12   | Orchestrator delivers itinerary to user.                                            |
| 13   | MCP Server monitors external APIs.                                                  |
| 14   | MCP detects event (flight delay) and pushes trigger to Message Bus.                 |
| 15   | Replanner consumes trigger and decides if replan is needed.                         |
| 16   | Replanner sends request to Optimizer for updated plan.                              |
| 17   | Optimizer returns revised plan.                                                     |
| 18   | Replanner forwards updated plan to Orchestrator.                                    |
| 19   | Orchestrator notifies user with revised itinerary.                                  |

---

## 6. PEAS Framework

| Component       | Description                                                                              |
| --------------- | ---------------------------------------------------------------------------------------- |
| **Performance** | Trip satisfaction, itinerary quality (comfort, cost, satisfaction), real-time adaptation |
| **Environment** | Weather, transport, events, user data APIs                                               |
| **Actuators**   | LLM Reasoner, Notification System, Optimizer Agent, Replanner Agent, Main Orchestrator   |
| **Sensors**     | Weather APIs, Transport Data, Event APIs, User Feedback                                  |

---

## 7. Agent Collaboration Model

* **Message Bus (Kafka/RabbitMQ)** enables asynchronous inter-agent communication.
* Each agent subscribes to specific topics (e.g., `weather.data`, `budget.data`).
* **Main Orchestrator** aggregates and coordinates all inter-agent messages.
* **Optimizer Agent** integrates agent results for final itinerary generation.

### Demo JSON Event and Partition Class

Partitioner class ensures each customer’s events are serialized to the same topic partition.

```json
{
  "Customer_ID": "",
  "Topic": "topic name",
  "from": "weather.data",
  "to": "replanner.do",
  "payload": {
    "confidence": "0.9"
  }
}
```

---

## 8. AI Layer

| Layer                  | Function                                     |
| ---------------------- | -------------------------------------------- |
| **Rule Layer**         | Constraints like budget and time             |
| **LLM Layer**          | Reasoning and summarization                  |
| **Optimization Layer** | Multi-objective trade-offs (cost vs comfort) |

---

## 9. Ethical & Responsible Considerations

| Principle          | Implementation                    |
| ------------------ | --------------------------------- |
| **Privacy**        | Encryption of sensitive data      |
| **Explainability** | Provide reasoned responses        |
| **Transparency**   | Include confidence scores         |
| **Accountability** | Enable human override and logging |

---

## 10. Future Enhancements

* **Predictive Forecasting**: Build ML models for improved forecasting or integrate third-party services.
* **GenAI Summarization**:

  * Suggest daily plans
  * Recommend items to carry
  * Suggest safe taxi services and highlight scams
  * Provide food, events, and local attraction insights

---

## 11. Evaluation Criteria

| Criteria            | Key Aspects                                                                               |
| ------------------- | ----------------------------------------------------------------------------------------- |
| **System Thinking** | Multi-agent hierarchical architecture, clear agent definitions, logical data/control flow |
| **Autonomy**        | Agents operate independently via Kafka                                                    |
| **Technical Depth** | Strong API, ML, and event-driven integration                                              |
| **Reasoning**       | Context-aware LLM, multi-objective optimization                                           |
| **Ethics & UX**     | Privacy, transparency, and personalization                                                |
