# 🎫 Enterprise Ticketing System (Conceptual)

## 📖 Application Overview
This repository contains the Business Analysis documentation and QA automation tests for a modular Incident Ticketing System. The system is designed to receive, route, and resolve incidents, orchestrating between multiple internal services and external systems via API.

## Core System Services
The architecture is divided into five distinct services:

*   **Incident Service (The Orchestrator):** The main brain of the system. Handles ticket initiation, data enrichment, classification routing, and overall status management.
*   **Workorder Service:** Manages the generation, assignment, and completion validation of field work (triggered only for specific ticket classifications).
*   **Scheduling Service:** Manages technician availability, generates schedule slots, and issues Booking IDs.
*   **Person Service (CRUD):** Manages user accounts, authentication, and Role-Based Access Control (RBAC) mappings (User Role ↔ User Group).
*   **Asset Service (CRUD):** Manages asset inventory and provides the critical `ownergroup` parameter required for ticket routing.


## Data Flow Diagram 
### Level 0
```mermaid
flowchart TD
    %% Simple, clean styles
    classDef external fill:#ebf8ff,stroke:#4299e1,stroke-width:2px,color:#2b6cb0
    classDef service fill:#e9d8fd,stroke:#805ad5,stroke-width:2px,color:#44337a

    %% Single External Entity
    ExtSys["External System"]:::external

    %% The 5 Internal Services
    subgraph CoreSystem ["Ticketing System"]
        Incident:::service
        Person:::service
        Asset:::service
        Scheduling:::service
        Workorder:::service
    end

    %% External calls the whole system boundary
    ExtSys -->|"API Calls"| CoreSystem
   
    %% Internal orchestration logic (Incident as the main brain)
    Incident <--> Person
    Incident <--> Asset
    Incident <--> Scheduling
    Incident --> Workorder
```
### Level 1 Incident Service
```mermaid
flowchart TD
    %% Styles
    classDef phase fill:#e2e8f0,stroke:#4a5568,stroke-width:2px,color:#2d3748,font-weight:bold
    classDef decision fill:#fefcbf,stroke:#d69e2e,stroke-width:2px,color:#744210
    classDef end_node fill:#f0fff4,stroke:#48bb78,stroke-width:2px,color:#276749
    classDef note fill:#fff5f5,stroke:#fc8181,stroke-width:2px,color:#c53030,font-style:italic

    %% The Major Phases
    P1["Ticket Initiation\n(External or Internal)"]:::phase
    P2["Data Enrichment\n(Call Asset Service)"]:::phase
    P3{"Classification\nCheck"}:::decision
   
    P4["Generate Workorder\n& Book Schedule"]:::phase
    P5["Skip WO/Schedule\n(Non-FISIK)"]:::phase
   
    P6["Ticket Active\n(Awaiting Action)"]:::phase
   
    P7{"Update Status\n(e.g., Close, Pending)"}:::decision
    P8(["Ticket Closed"]):::end_node

    %% The Flow
    P1 --> P2 --> P3
    P3 -->|"FISIK"| P4
    P3 -->|"Non-FISIK"| P5
    P4 --> P6
    P5 --> P6
    P6 --> P7
    P7 -->|"Validation Passed"| P8
    P7 -->|"Validation Failed"| Reject(["Reject: Access Denied"]):::end_node

    %% Global Rule Note
    subgraph GlobalRules ["⚠️ Global System Rule"]
        direction LR
        GR["ALL change actions (Creation, Assignment, Closure) require:\n1. User Role Validation\n2. Business Logic Checks"]:::note
    end
   
    GlobalRules -.- P6
```
### Level 1 Scheduling Service
```mermaid
flowchart TD
    %% Styles
    classDef phase fill:#e2e8f0,stroke:#4a5568,stroke-width:2px,color:#2d3748,font-weight:bold
    classDef end_node fill:#f0fff4,stroke:#48bb78,stroke-width:2px,color:#276749
    classDef external fill:#ebf8ff,stroke:#4299e1,stroke-width:2px,color:#2b6cb0

    %% The Scheduling Process
    P1["Generate Available Schedules\n(for Technicians)"]:::phase
    
    P2["Schedule Slots Active"]:::phase
    
    P3["Booking Request\n(from External System)"]:::external
    
    P4{"BookingID Available?"}:::decision
    
    P5["Book Schedule\n(Book Booking ID)"]:::phase
    
    P6(["Schedule Status: BOOKED"]):::end_node
    P7(["Reject: Slot Unavailable"]):::end_node

    %% The Flow
    P1 --> P2 --> P3 --> P4
    P4 -->|"Yes"| P5 --> P6
    P4 -->|"No"| P7
```
### Level 1 Workorder Service
```mermaid
flowchart TD
    %% Styles
    classDef phase fill:#e2e8f0,stroke:#4a5568,stroke-width:2px,color:#2d3748,font-weight:bold
    classDef decision fill:#fefcbf,stroke:#d69e2e,stroke-width:2px,color:#744210
    classDef end_node fill:#f0fff4,stroke:#48bb78,stroke-width:2px,color:#276749
    classDef external fill:#ebf8ff,stroke:#4299e1,stroke-width:2px,color:#2b6cb0

    %% The Workorder Process
    P1["Workorder Creation Request\n(Incident Service)"]:::phase
    
    P2{"Requires\nScheduling?"}:::decision
    
    %% Path A: Without Scheduling
    P3A["Create Workorder\n(Ticket ID - WO Number)"]:::phase
    
    %% Path B: With Scheduling
    P3B["Get Booking ID from Scheduling Service"]:::phase
    P4B["Create Workorder with Booking ID\n(Ticket ID - WO Number - Booking ID)"]:::phase
    P5B["Book Schedule"]:::phase
    
    %% Common Path: Status Update
    P6["Workorder Active"]:::phase
    
    P7["Completion Request\n(via Incident Service)"]:::external
    
    P8{"Validate Completion\nLogic"}:::decision
    
    P9(["Workorder Status: COMPLETE"]):::end_node
    P10(["Reject: Validation Failed"]):::end_node

    %% The Flow
    P1 --> P2
    P2 -->|"No"| P3A
    P2 -->|"Yes"| P3B
    P3A --> P6
    P3B --> P4B --> P5B --> P6
    P6 --> P7 --> P8
    P8 -->|"OK"| P9
    P8 -->|"Failed"| P10
```

