# Overview
This repository demonstrates how Vonage Network APIs and Vonage Video API can be combined to create a network-powered, passwordless onboarding experience for real-time video applications.  

Using Silent Authentication (Silent Auth) via the Verify API with application-level SMS fallback, the app verifies device ownership before generating a Video API token and joining a live session.  

- Vonage APIs  
  - Verify API: Device ownership validation using Silent Auth / SMS fallback    
  - Video API: Real-time video creation and access  

For the architectural decisions and product rationale behind this demo, see the accompanying blog post:  

## Table of Contents  
- [Architecture](#Architecture)
- [Prerequisites](#Prerequisites)
  - [Notes](#notes)
- [Setup & Run for Development](#setup--run-for-development)
  - [Application Setup](#application-setup)
  - [Backend Setup](#backend-setup)
  - [Android Setup](#android-setup)
- [Error Handling](#error-handling)
- [Failure Scenarios](#failure-scenarios)
- [Production Recommendations](#production-recommendations)

# Architecture  
The solution is intentionally designed with a clear separation of concerns. Authentication and video are deliberately decoupled. Verification completes first, only then is a Video token issued and the session initialized.  
- Android Client
  - Collects the phone number
  - Routes based on verification channel (silent_auth or sms)
  - Forces the Silent Auth request over the mobile network using the library
  - Joins the Video session only after verification is completed
- Python Backend (FastAPI)
  - Initiates verification via the Verify API
  - Treats Silent Auth support as a routing decision
  - Confirms verification
  - Generates a Video API token

# Prerequisites  
To run this demo, you will need:
- A Vonage API account
- Python<=3.9
- Android Studio
- A physical device with mobile data enabled SIM

## Notes  
- Video works with an OpenTok account, but Verify requires a Vonage API account.
- The Android emulator can be used for basic UI testing using a Virtual Operator. 
- To test Silent Auth, you have two options:
  - Use the Virtual Operator provided for testing purposes
  - Use a SIM from a supported mobile operator with mobile data enabled
- For testing, a supported phone number must be registered in the Network Registry Playground.
- In a production environment, registration with the Network Registry is required.

# Setup & Run for Development
## 1. Application Setup
Before running the demo, make sure your Vonage application is properly configured:
  1. Log in to the Vonage API Dashboard.
  2. Create or open an existing Application.
  3. Enable:
    - Video 
    - Network Registry with “Playground” access type
  4. Click on “Configure Playground”
  5. From the  “Add numbers” , Add a phone number from supported operators that you’d like to test Silent Auth

## 2. Backend Setup
  1. Run the following commands under the backend folder:  
```
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```
  2. Rename the .env.example to .env and add your credentials:
```
VONAGE_APPLICATION_ID=application id
VONAGE_PRIVATE_KEY_PATH=path to private.key
VONAGE_VIDEO_SESSION_ID=video session id
```
For test purposes, a fixed session ID can be generated via the Video Playground from https://tools.vonage.com/video/playground.  

  3. Run the backend:
```
uvicorn app:app --reload --host 0.0.0.0 --port 8000
```
If testing on a physical device within the same network, obtain your machine’s local IP address (e.g., ipconfig getifaddr en0 on macOS) and update the base URL accordingly (e.g., http://192.168.1.193:8000). Alternatively, expose the backend using ngrok.  

## 3. Android Setup
  1. Open the client/android project in Android Studio
  2. Update the backend base URL in ServerConfig.kt
  3. Run on a physical device

# Error Handling
This demo is designed to fail safely without crashing, and to keep users moving through the flow with clear recovery paths.
Principles  
- No app-killing failures for recoverable errors  
Network issues, unsupported carriers, invalid inputs, and unexpected responses are handled in-place with user-facing guidance.  
- UI always returns to an actionable state  
Every failure path unlocks the relevant UI (buttons re-enabled) and returns focus to the next expected action.  
- Errors are routed, not treated as “exceptions”  
Silent Auth “not supported” is handled as a routing outcome (SMS fallback), not a hard failure.  

# Failure Scenarios
- Input validation  
Invalid phone numbers / SMS codes are blocked client-side with field-level errors and focus.  
- Silent Auth check failures  
If the cellular request fails (or returns an error payload), the app does not close—it unlocks the UI and prompts the user to retry (or proceed via SMS when applicable).  
- Verification confirmation failures  
Non-200 responses or non-completed statuses keep the user in the authentication flow, with a retry path.  
- Video token failures  
Token fetch errors are treated as recoverable network/setup issues: the UI is unlocked and the user is prompted to try again (video is not started in a partial state).

# Production Recommendations
- Create sessions dynamically (per user, per room, or per meeting) and issue short-lived tokens scoped to that session.  
- Replace generic message like “Silent Auth failed” with action-based and user-friendly messages (“Out of range, check your mobile network and try again”, “It looks like you have multiple SIMs. Make sure mobile data is enabled for this number and try again.”).  
- Add retry limits + backoff for transient network errors.  
- Log full technical details to telemetry, but show minimal user text.  
- Provide a single “Start over” CTA that resets state (verifyRequestId, verifyCheckUrl, UI mode, locks).  









