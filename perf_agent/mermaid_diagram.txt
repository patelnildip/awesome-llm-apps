sequenceDiagram
    autonumber
    actor User as 👤 User
    participant UI as 💻 Streamlit (Frontend)
    participant API as ⚙️ FastAPI (Backend)
    participant Crew as 🤖 CrewAI (Agents)
    participant FS as 📂 File System
    participant JMeter as 🚀 Apache JMeter
    participant Target as 🌐 Target Website

    User->>UI: Enter URL, Users, Duration & Click "Start"
    UI->>API: POST /start-test (JSON payload)
    
    Note over API, Crew: Handover to AI Agents
    API->>Crew: Kickoff Agents (Architect & Runner)
    
    %% Phase 1: Script Generation
    rect rgb(240, 248, 255)
        Note right of Crew: Agent 1: QA Architect
        Crew->>Crew: Generate JMX XML Logic
        Crew->>FS: Save "test_script.jmx"
    end

    %% Phase 2: Execution
    rect rgb(255, 248, 240)
        Note right of Crew: Agent 2: Performance Analyst
        Crew->>JMeter: Execute Command (jmeter -n -t ...)
        JMeter->>Target: HTTP Requests (Load Test)
        Target-->>JMeter: HTTP Responses
        JMeter->>FS: Write "results.csv"
    end

    %% Phase 3: Analysis
    rect rgb(240, 255, 240)
        Crew->>FS: Read "results.csv"
        FS-->>Crew: Return CSV Data
        Crew->>Crew: Analyze Latency & Errors using LLM
    end

    Crew-->>API: Return Text Summary (Report)
    API-->>UI: Return JSON Response
    UI-->>User: Display Success Message & Report
