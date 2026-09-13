# Broker CALL-E Agent

A LangGraph-based real estate broker assistant that uses CALL-E to make natural phone calls and gather property requirements from customers.

## Architecture

```text
Phone Number + Call Objective
          │
          ▼
     LangGraph
          │
          ▼
   CALL-E Call Node
          │
          ▼
   Real Person Call
          │
          ▼
  Natural Conversation
          │
          ▼
  Gather Requirements
          │
          ▼
  Structured JSON
          │
          ▼
  Save to Database
          │
          ▼
        END
```

The CALL-E call node uses the **official CALL-E Python SDK** (`calle-ai`):

```python
from calle import CalleClient

client = CalleClient(api_key=os.environ["CALLE_API_KEY"])
call = client.calls.create_and_wait(
    task="Call <E164_PHONE> and ask whether they can hear clearly.",
    result_schema={...},
)
```

The destination phone number is passed directly in the task, in E.164 format (e.g. `+919XXXXXXXXX`). No phone-number purchase or extra telephony provider is required by the SDK integration itself. CALL-E dashboard number provisioning / caller-ID / KYC, if required for your account, is an account-side setup concern surfaced by CALL-E at call time (e.g. `forbidden`, `unsupported_region`).

## Project Structure

```text
broker-call-agent/
│
├── app/
│   ├── main.py
│   │
│   ├── graph/
│   │   ├── state.py
│   │   ├── nodes.py
│   │   └── workflow.py
│   │
│   ├── services/
│   │   ├── calle.py
│   │   └── storage.py
│   │
│   └── schemas.py
│
├── data/
│   └── calls.json
│
├── .env
├── requirements.txt
└── README.md
```

## Setup

1. Install dependencies:

```bash
pip install -r requirements.txt
```

2. Set up environment variables:

```bash
cp .env .env.local
# Edit .env.local with your CALLE_API_KEY
```

3. Run the server:

```bash
uvicorn app.main:app --reload
```

4. Access the API docs:

```text
http://localhost:8000/docs
```

## API Usage

### Make a Call

**POST** `/call`

**Request:**

```json
{
    "phone_number": "+91XXXXXXXXXX",
    "objective": "Understand the customer's requirements for buying a residential property."
}
```

**Response:**

```json
{
    "success": true,
    "call_status": "completed",
    "gathered_information": {
        "intent": "buy",
        "location": "Hinjewadi",
        "property_type": "2BHK",
        "budget": "80 lakh",
        "bedrooms": "2",
        "timeline": "within 3 months",
        "additional_requirements": "Parking required",
        "evidence_summary": "The customer asked for a 2BHK in Hinjewadi under 80 lakh and wants possession within 3 months."
    }
}
```

On failure (`success: false`) an `error` field explains why (invalid phone, API error, timeout, or call not completed).

## How It Works

1. **validate_input** - Validates phone number is provided and in E.164 format
2. **prepare_call** - Sets up the call objective
3. **call_e** - Initiates CALL-E phone call with natural conversation (official SDK, waits for structured result)
4. **check_result** - Never assumes success: only treats the call as successful when `status == "completed"`, `task_completed == true`, and `structured_result` is a valid object
5. **store_result** - Saves structured results to JSON storage (only reached on success)
6. **handle_failure** - Records why the call failed and ends without storing

## Data Storage

Calls are stored in `data/calls.json`:

```json
[
    {
        "timestamp": "2026-08-08T16:30:00",
        "phone_number": "+91XXXXXXXXXX",
        "objective": "Understand the customer's requirements.",
        "call_status": "completed",
        "completion_confidence": {
            "score": 0.94,
            "label": "high"
        },
        "gathered_information": {
            "intent": "buy",
            "location": "Hinjewadi",
            "property_type": "2BHK",
            "budget": "80 lakh",
            "bedrooms": "2",
            "timeline": "within 3 months",
            "additional_requirements": "Parking required",
            "evidence_summary": "The customer asked for a 2BHK in Hinjewadi under 80 lakh and wants possession within 3 months."
        }
    }
]
```

## Tests

Minimal SDK test (official quickstart flow, one call):

```bash
python test_sdk_minimal.py
```

Broker call via the reusable `make_broker_call` service:

```bash
python test_sdk_broker.py
```

Both require `CALLE_API_KEY` and a test phone `CALLE_TEST_PHONE` (E.164) in `.env`.

## Important Notes

* Only call people who have given appropriate permission/authorization
* Identify the AI/automated nature of the call as required by applicable law
* Don't design the agent to impersonate a human or conceal that it's automated
* Requires a valid CALL-E API key

## Future Versions

* **Version 2**: LLM validation + PostgreSQL/Supabase
* **Version 3**: Customer database + Property matching + Broker dashboard
* **Version 4**: RAG + Memory + Advanced property matching
