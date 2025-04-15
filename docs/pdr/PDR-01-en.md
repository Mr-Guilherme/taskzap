# Product Design Review (PDR)  
**Project:** WhatsApp Task Bot with OpenAI and Google Calendar  
**Date:** 12/04/2025  
**Author:** Guilherme Rodrigues

---

## 1. Product Overview

The WhatsApp Task Bot is an application that allows users to send audio messages to a WhatsApp number. The application listens to the audio, transcribes it using OpenAI's Whisper model, extracts tasks using a language model (GPT), sends the tasks back to the user for confirmation via WhatsApp, and after confirmation, adds the events to Google Calendar.

This system is ideal for users who prefer voice-based organization, focusing on quick productivity via mobile and seamless calendar integration.

---

## 2. Target Audience

- Professionals who use WhatsApp as their main communication channel  
- Users who prefer voice input  
- People who need to organize routines and appointments quickly  

---

## 3. Project Objectives

- Reduce friction in task organization through voice commands  
- Automate the flow: voice → text → tasks → calendar  
- Use accessible and popular tools (WhatsApp, Google Calendar)  
- Minimize costs and infrastructure complexity  

---

## 4. Architecture and Components

### 4.1 Technology Stack

| Component         | Tool / Libraries                    |
|--------------------|--------------------------------------|
| Backend            | Node.js + NestJS                     |
| WhatsApp Bot       | [Baileys](https://github.com/WhiskeySockets/Baileys) |
| Audio Transcription | OpenAI Whisper (REST API via `fetch`) |
| Task Extraction    | GPT-3.5 (REST API via `fetch`)       |
| Calendar Integration | Google Calendar API                 |
| Infrastructure     | Docker, VPS, GitHub Actions CI/CD   |
| Storage            | Supabase (self-hosted)              |
| Monitoring         | Uptime Kuma (self-hosted)           |

---

### 4.2 Data Flow

1. User sends an audio message to the bot's WhatsApp number  
2. Bot listens with Baileys and downloads the audio locally  
3. Audio is converted via FFmpeg and sent to OpenAI Whisper  
4. Transcribed text is sent to GPT-3.5 for task extraction  
5. Bot responds with a formatted list of tasks on WhatsApp  
6. User confirms or edits tasks via message  
7. Confirmed events are inserted into Google Calendar  
8. Original audio is deleted from the system after processing  

---

### 4.3 Component Diagram

```text
WhatsApp User
     |
     v
  [Baileys Bot]
     |
     v
[Transcription (Whisper API)]
     |
     v
[Task Extraction (GPT)]
     |
     v
[WhatsApp Confirmation]
     |
     v
[Google Calendar API]
```

---

### 4.4 Google Calendar Authentication Flow

The bot uses Google's OAuth2 flow to obtain calendar access permission for each user individually. After authorization, tokens are saved in Supabase (self-hosted), associated with the user's WhatsApp phone number.

#### Flow steps:

1. Bot receives an audio message
2. Checks if the WhatsApp number already has a token saved in Supabase
3. If no token exists, sends a Google Calendar authorization link
4. User clicks, authorizes, and Google redirects to the server
5. Server exchanges `code` for `access_token` and `refresh_token`
6. Tokens are saved in Supabase, linked to the WhatsApp number
7. With valid permission, the bot can create events in the user's Google Calendar

---

### 4.5 Supabase (self-hosted)

#### Intended use:

- Storage of authentication tokens per user
- Relationship with WhatsApp number
- Allows token reuse for future calls

#### `google_tokens` Table

| Field            | Type       | Notes                           |
|------------------|------------|----------------------------------------|
| `whatsapp_number`| TEXT       | User key (WhatsApp number)     |
| `access_token`   | TEXT       | Temporary access token             |
| `refresh_token`  | TEXT       | Used to renew the access_token      |
| `expires_at`     | TIMESTAMP  | When the access_token expires           |
| `google_email`   | TEXT       | (Optional) Google account email       |
| `created_at`     | TIMESTAMP  | Record creation date            |

> **Security**: tokens must be encrypted in the backend before saving.

---

### 4.6 Updated Diagram

```text
WhatsApp User
     |
     v
  [Baileys Bot]
     |
     v
  [NestJS Server]
     |      \
     |       -----> [Check tokens in Supabase]
     |                    |
     |                [Has token?]
     |                 /      \
     |          [Yes]         [No]
     |             |              \
     |       Create event        Send OAuth link
     |                            |
     |                    User authorizes
     |                            |
     |                 /auth/callback receives code
     |                            |
     |                Exchange for token, save in Supabase
     |                            |
     |                    Normal flow continues
     |
     v
[OpenAI APIs] → Transcription + GPT
     |
     v
[Google Calendar API]
```

---

## 5. Infrastructure

- **Hosting**: VPS (1 vCPU, 2GB RAM)
- **Deployment**: Docker + GitHub Actions
- **Supabase**: self-hosted instance, with local Postgres database
- **Monitoring**: Uptime Kuma (for health, HTTP, WebSocket)

---

## 6. Security

- Audio files are kept only locally and deleted after processing
- Sensitive tokens are encrypted before storage
- API communication via HTTPS
- WhatsApp sessions stored securely

---

## 7. Estimated Costs

| Item                | Cost per User | Total Cost (1,000 users) |
|---------------------|------------------|-------------------------------|
| OpenAI Whisper      | $0.54            | $540                          |
| OpenAI GPT-3.5      | $0.06            | $60.75                        |
| Google Calendar API | $0.00            | $0                            |
| VPS + Infrastructure| —                | ~$10                          |
| Supabase (self-hosted)| —              | ~$0 (included in VPS)       |
| Monitoring          | —                | ~$5                           |
| **Total**           | **~$0.60**       | **~$615.75/month**              |

---

## 8. Initial Roadmap

| Phase                              | Estimated Date       |
|-----------------------------------|---------------------|
| MVP (transcription + simple response) | April 2025         |
| Google authentication per user     | April/May 2025     |
| Task confirmation via WhatsApp     | May 2025          |
| Supabase integration               | May 2025           |
| Google Calendar integration        | May/June 2025     |
| Token/login panel (optional)       | June 2025         |
| Public launch (beta)               | July 2025         |

---

## 9. Risks and Mitigations

| Risk                                  | Mitigation                                           |
|----------------------------------------|-----------------------------------------------------|
| Baileys is an unofficial library       | Active monitoring and fallback to Twilio if needed |
| Variable costs with OpenAI             | Rate limiting, partial caching                     |
| Incorrect task interpretation          | Manual confirmation on WhatsApp before calendar submission |
| WhatsApp session drop                  | Automatic reauthentication and session backup       |
| OAuth token expiration                 | Automatic renewal with `refresh_token`           |

---

## 10. Conclusion

This project combines accessible and low-cost technologies to create a virtual task assistant via WhatsApp. With a pragmatic approach (monolith, REST, Docker), it focuses on delivering value quickly with a good user experience.

--- 