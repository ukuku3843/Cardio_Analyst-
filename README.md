# Cardio Analyst — Microservice Setup Guide

## Architecture

```text
┌─────────────────────┐        POST /analyze          ┌───────────────────────────┐
│   Java App          │ ──── 5s Telemetry Chunks ───▶ │  Flask Microservice       │
│   (Test.java)       │                               │  (cardio_service.py)      │
│                     │ ◀─── Risk Assessment JSON ─── │  - Manages Patient State  │
│   Control_Station   │                               │  - Calls Llama 3 Engine   │
│   .displayAlert()   │                               │  - Serves HTML Dashboard  │
└─────────────────────┘                               └───────────────────────────┘
                                                                │
                                                      GET /api/dashboard_data
                                                                ▼
                                                      ┌───────────────────────────┐
                                                      │  Live Web Dashboard       │
                                                      │  (index.html)             │
                                                      └───────────────────────────┘
Step 0 - Download
Downlad the cardio_an zip and unpack the files

Step 1 — Set Up the Python Microservice
Install dependencies
```Bash
    pip install -r requirements.txt
    (Make sure your requirements.txt includes flask and your specific Llama 3 integration library, such as openai or groq)

Download and install Llama 3 (Local Setup via Ollama)
If you are running Llama 3 locally:

    Download Ollama from ollama.com and install it.

Open your terminal and run:

```Bash
    ollama run llama3
        (Note: If you are using a cloud provider like Groq instead, skip this step and simply add your API key to a .env file).
```
Start the Flask service
    Bash
    python cardio_service.py
You should see:

Plaintext
╔══════════════════════════════════════════════╗
║  Cardio Analyst: Streaming Monitor Active  🫀  ║
║  Listening on http://localhost:5000          ║
╚══════════════════════════════════════════════╝
    Verify it's running
    Open your web browser and navigate to http://localhost:5000 to view the Live ICU Dashboard.

Step 2 — Run the Java App
    Make sure the Flask service is running FIRST, then compile and run:

Bash
    # Compile (adjust classpath for gson jar)
    javac -cp ".:gson-2.10.1.jar" Test.java

# Run
    java -cp ".:gson-2.10.1.jar" Test
    Message Flow (Per Admitted Patient)
    The system uses a stateful "Chunk & Compare" streaming architecture.

Java Data Generation: Java generates a 3-minute continuous waveform and begins streaming it in 5-second TelemetryPacket chunks to POST /analyze.

Phase 1 (Baseline Calibration): - For the first 30 seconds, Flask catches the incoming 5-second chunks and stores them.

Once 30 seconds of data is accumulated, Flask sends the bulk data to Llama 3 to establish a personalized clinical baseline.

Phase 2 (Active Monitoring):

As subsequent 5-second chunks arrive, Flask sends the new data alongside the established baseline to Llama 3.

Llama 3 returns a structured JSON risk assessment comparing the current state to the baseline:

JSON
{
  "patient_id": 1042,
  "risk": true,
  "level": "CRITICAL",
  "assessment": "Heart rate has deviated 40 BPM above the established baseline.",
  "action": "Notify attending physician and prepare for Code Blue.",
  "timestamp": "2026-05-12T14:33:01Z"
}
System Alerts: - Java reads risk: true → calls Control_Station.displayAlert() and prints to the console.

The HTML Dashboard simultaneously polls the Flask server and visually triggers the global Code Blue banner.

Endpoints
Method    Path    Description
GET    /    Serves the live HTML ICU Dashboard
GET    /api/dashboard_data    Polled every 2 seconds by the UI to fetch active patient states
GET    /patient/<id>    Looks up patient vital profiles from the CSV directory
POST    /analyze    Accepts streaming 5-second JSON chunks; routes to Llama 3
