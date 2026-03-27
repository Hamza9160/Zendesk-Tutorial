# Zendesk - Technical Documentation

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Zendesk User Setup (Prerequisite)](#zendesk-user-setup-prerequisite)
4. [JWT Token Management (Single Source of Truth)](#jwt-token-management-single-source-of-truth)
5. [Screen Flow Diagram](#screen-flow-diagram)
6. [Submit Button - Complete API Flow](#submit-button---complete-api-flow)
7. [API Retry System](#api-retry-system)
8. [Zendesk Logout](#zendesk-logout)
9. [Screens & Components](#screens--components)
10. [Ticket Status Filtering](#ticket-status-filtering)
11. [API Endpoints](#api-endpoints)
12. [Error Handling](#error-handling)

---

## Overview

The **Concierge** feature enables prescribers to submit requests directly from the mobile app. It integrates with **Zendesk Sunshine Conversations** for ticket creation and real-time messaging.

### Key Capabilities

| Feature | Description |
|---------|-------------|
| Request Free Samples | Request pharmaceutical samples |
| Schedule Rep Visit | Request a visit from a pharmaceutical representative |
| Custom Request | Submit any custom request (clinical trials, medication savings, etc.) |
| Request Tracking | View active and archived requests |
| Real-time Messaging | Continue conversations via Zendesk messaging |
| Contact Preferences | Set preferred contact hours |

---

## Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                            MVVM ARCHITECTURE                              │
└───────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│      VIEW       │     │   VIEWMODEL     │     │     MODEL       │
│   (Composable)  │────▶│  (StateFlow)    │────▶│   (Repository)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        │   observes state      │   manages state       │   API calls
        │◀──────────────────────│◀──────────────────────│
        │                       │                       │
┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
│   Concierge   │       │   Concierge   │       │  HTTP Client  │
│   Screens     │       │   ViewModels  │       │  (Network)    │
└───────────────┘       └───────────────┘       └───────────────┘
```

### Component Diagram

```
┌───────────────────────────────────────────────────────────────────────────┐
│                           CONCIERGE MODULE                                │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                            UI LAYER                                 │  │
│  │                                                                     │  │
│  │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │  │
│  │   │ Concierge       │  │ Request List    │  │ Archived        │    │  │
│  │   │ Screen          │  │ Screen          │  │ Screen          │    │  │
│  │   │                 │  │                 │  │                 │    │  │
│  │   │ • Request Types │  │ • New/Open/     │  │ • Closed        │    │  │
│  │   │ • Text Input    │  │   Pending only  │  │ • Solved        │    │  │
│  │   │ • Submit Button │  │ • Tap to open   │  │ • Hold          │    │  │
│  │   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │  │
│  │            │                    │                    │              │  │
│  │   ┌────────┴────────┐  ┌────────┴────────────────────┴────────┐    │  │
│  │   │ Settings Screen │  │       Shared UI Components           │    │  │
│  │   │ • Preferences   │  │ • RequestCard • RadioButton • Dialogs│    │  │
│  │   └─────────────────┘  └──────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                     │                                     │
│                                     ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                         VIEWMODEL LAYER                             │  │
│  │                                                                     │  │
│  │   ┌─────────────────────────┐   ┌─────────────────────────┐        │  │
│  │   │   ConciergeViewModel    │   │  RequestsListViewModel  │        │  │
│  │   │                         │   │                         │        │  │
│  │   │ • Form validation       │   │ • Fetch active requests │        │  │
│  │   │ • API state machine     │   │ • Fetch archived reqs   │        │  │
│  │   │ • Submission flow       │   │ • Search by user email  │        │  │
│  │   └─────────────────────────┘   └─────────────────────────┘        │  │
│  │                                                                     │  │
│  │   ┌───────────────────────────────────────────────────────────┐    │  │
│  │   │                     SharedViewModel                       │    │  │
│  │   │  • User Profile • JWT Token • Conversation ID • Test NPI  │    │  │
│  │   └───────────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                     │                                     │
│                                     ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                          DATA LAYER                                 │  │
│  │                                                                     │  │
│  │   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐  │  │
│  │   │   HTTP Client   │   │  Zendesk SDK    │   │ Local Storage   │  │  │
│  │   │   (Network)     │   │  (Messaging)    │   │ (SharedPrefs)   │  │  │
│  │   └─────────────────┘   └─────────────────┘   └─────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Zendesk User Setup (Prerequisite)

Before a user can submit requests, they must be registered with Zendesk. This happens automatically **right after user login** on the Home Page, not when opening the Concierge feature.

### User Creation Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 ZENDESK USER SETUP FLOW (PREREQUISITE)                  │
└─────────────────────────────────────────────────────────────────────────┘

    USER LOGS IN → HOME PAGE LOADS
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   STEP 1: CREATE ZENDESK SUPPORT USER                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Endpoint: POST {BASE_URL}/api/v2/users.json                            │
│  Auth: Basic Auth (email/token:apiToken)                                │
│                                                                         │
│  Request Body:                                                          │
│  {                                                                      │
│    "user": {                                                            │
│      "name": "{firstName}",                                             │
│      "email": "{userEmail}",                                            │
│      "external_id": "{npi}"                                             │
│    }                                                                    │
│  }                                                                      │
│                                                                         │
│  Purpose: Create user account in Zendesk Support for ticket management │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                STEP 2: CREATE SUNSHINE CONVERSATIONS USER               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Endpoint: POST {BASE_URL}/sc/v2/apps/{appId}/users                     │
│  Auth: Basic Auth (appKey:appSecret)                                    │
│                                                                         │
│  Request Body:                                                          │
│  {                                                                      │
│    "externalId": "{npi}",                                               │
│    "identities": [                                                      │
│      { "type": "email", "value": "{userEmail}" }                        │
│    ],                                                                   │
│    "profile": {                                                         │
│      "givenName": "{firstName}",                                        │
│      "surname": "{lastName}",                                           │
│      "email": "{userEmail}"                                             │
│    }                                                                    │
│  }                                                                      │
│                                                                         │
│  Purpose: Create user in Sunshine Conversations for real-time messaging│
│                                                                         │
│  Response Handling:                                                     │
│  • 201 Created → User created successfully, returns userId              │
│  • 409 Conflict → User already exists, fetch existing user              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      STEP 3: GET OR CREATE JWT TOKEN                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Uses: sharedViewModel.getOrCreateJwtToken()                            │
│                                                                         │
│  • Returns cached JWT if available                                      │
│  • Generates new JWT if cache is empty                                  │
│                                                                         │
│  JWT Claims:                                                            │
│  {                                                                      │
│    "scope": "user",                                                     │
│    "external_id": "{npi}",                                              │
│    "name": "{fullName}",                                                │
│    "email": "{userEmail}",                                              │
│    "iat": {issuedAt},                                                   │
│    "exp": {expiresAt}  // 6 hours from generation                       │
│  }                                                                      │
│                                                                         │
│  Purpose: Authenticate user with Zendesk Messaging SDK                  │
│  Token Expiry: 6 hours                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    STEP 4: LOGIN TO ZENDESK SDK                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Uses the generated JWT token to authenticate with Zendesk Messaging    │
│                                                                         │
│  Zendesk.instance.loginUser(jwt = jwtToken)                             │
│                                                                         │
│  On Success:                                                            │
│  • Mark user as logged in                                               │
│  • Push FCM token for push notifications                                │
│                                                                         │
│  On Failure:                                                            │
│  • Log error for debugging                                              │
│  • User can still use other app features                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
           │
           ▼
    USER IS NOW READY TO SUBMIT REQUESTS
```

### User Identification

The **NPI (National Provider Identifier)** is used as the `external_id` to link the app user with their Zendesk account. This allows:

- Consistent user identification across platforms
- Ticket attribution to the correct prescriber
- Conversation history preservation

### Handling Existing Users

When creating a Sunshine user, if the API returns **409 Conflict**, the system automatically:

1. Fetches the existing user by external ID
2. Retrieves the existing user's Sunshine ID
3. Continues with the normal flow

```
GET {BASE_URL}/sc/v2/apps/{appId}/users?externalId={npi}
```

### Error Handling

| Step | Error Condition | Handling |
|------|-----------------|----------|
| Create Zendesk User | API failure (-1) | Log error, continue with flow |
| Create Sunshine User | 409 Conflict | Fetch existing user by external ID |
| Create Sunshine User | Other errors | Log error, continue with flow |
| Generate JWT | Missing user data | Skip JWT generation |
| Zendesk SDK Login | SDK not initialized | Retry up to 10 times (500ms delay) |
| Zendesk SDK Login | Login failure | Log error, user can still use app |

---

## JWT Token Management (Single Source of Truth)

The app implements a **single source of truth** pattern for JWT token management. Instead of generating new JWT tokens at multiple places throughout the app, all JWT operations go through `SharedViewModel.getOrCreateJwtToken()`.

### Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                     JWT TOKEN MANAGEMENT                                  │
└───────────────────────────────────────────────────────────────────────────┘

                        SharedViewModel
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        getOrCreateJwtToken()  setJwtToken()  clearJwtToken()
              │                    │               │
              │                    │               │
    ┌─────────┴─────────┐         │               │
    │                   │         │               │
    ▼                   ▼         │               │
  JWT exists?        No JWT       │               │
    │                   │         │               │
    │ Yes               │         │               │
    │                   ▼         │               │
    │           generateTestJwt() │               │
    │                   │         │               │
    │                   └─────────┘               │
    │                       │                     │
    ▼                       ▼                     ▼
  Return               Store & Return         Clear JWT
  existing JWT         new JWT               (on logout)
```

### Implementation

```
fun getOrCreateJwtToken(): String {
    val existingJwt = _jwtToken.value
    if (!existingJwt.isNullOrEmpty()) {
        return existingJwt
    }
    
    val profile = getProfile.value
    val npi = profile?.npi?.toString() ?: ""
    val fullName = "${profile?.firstname ?: ""} ${profile?.lastname ?: ""}".trim()
    val email = profile?.email ?: ""
    
    return generateTestJwt(
        externalId = npi,
        name = fullName,
        email = email,
        sharedViewModel = this
    )
}

fun clearJwtToken() {
    _jwtToken.value = null
}
```

### Usage Pattern

All screens and ViewModels use `getOrCreateJwtToken()` instead of calling `generateTestJwt()` directly:

```
val jwtToken = sharedViewModel.getOrCreateJwtToken()
```

### JWT Lifecycle

| Event | Action |
|-------|--------|
| First Zendesk operation | `getOrCreateJwtToken()` generates and caches JWT |
| Subsequent operations | Returns cached JWT (no regeneration) |
| User signs out | `clearJwtToken()` removes cached JWT |
| Account deactivation | `clearJwtToken()` removes cached JWT |
| Next login | Fresh JWT generated on first Zendesk operation |

### Benefits

1. **Consistency** - Same JWT used across all Zendesk operations
2. **Performance** - Avoids redundant JWT generation
3. **Reliability** - Eliminates sync issues from multiple tokens
4. **Maintainability** - Single place to modify JWT logic

---

## Screen Flow Diagram

```
                                    ┌─────────────────┐
                                    │   Home Page     │
                                    │   (Concierge    │
                                    │    Tab)         │
                                    └────────┬────────┘
                                             │
                                             ▼
                              ┌──────────────────────────┐
                              │    Concierge Screen      │
                              │                          │
                              │  ┌────────────────────┐  │
                              │  │ Request Samples ○  │  │
                              │  │ Schedule Rep    ○  │  │
                              │  │ Custom Request  ○  │  │
                              │  └────────────────────┘  │
                              │                          │
                              │  [Message Icon] [Submit] │
                              └──────────┬───────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
         ┌──────────────────┐  ┌─────────────────┐  ┌─────────────────┐
         │  Zendesk         │  │ Requests List   │  │ Settings Menu   │
         │  Messaging       │  │ Screen          │  │                 │
         │  (after submit)  │  │                 │  │ • Preferences   │
         └──────────────────┘  └────────┬────────┘  │ • Archived      │
                                        │           └────────┬────────┘
                          ┌─────────────┴─────────────┐      │
                          │                           │      │
                          ▼                           ▼      ▼
               ┌─────────────────┐           ┌─────────────────┐
               │ Tap Request     │           │ Contact         │
               │ Card            │           │ Preferences     │
               │                 │           │                 │
               │ Opens Zendesk   │           │ • Hours to Call │
               │ Messaging       │           │ • Do Not Contact│
               └─────────────────┘           └─────────────────┘
                                                     │
                                                     ▼
                                            ┌─────────────────┐
                                            │ Archived        │
                                            │ Requests        │
                                            │                 │
                                            │ View non-active │
                                            │ conversations   │
                                            │ (closed, solved,│
                                            │ pending, hold)  │
                                            └─────────────────┘
```

---

## Submit Button - Complete API Flow

When user clicks the **Submit** button, the following sequence executes:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    SUBMIT BUTTON - COMPLETE API FLOW                      │
└───────────────────────────────────────────────────────────────────────────┘

    USER CLICKS SUBMIT
           │
           ▼
    ┌──────────────┐
    │  VALIDATION  │
    │              │
    │ 1. Request   │
    │    type      │
    │    selected? │
    │              │
    │ 2. Message   │
    │    not empty?│
    │              │
    │ 3. NPI       │
    │    present?  │
    │              │
    │ 4. Phone     │
    │    present?  │
    └──────┬───────┘
           │
           │ All Valid
           ▼
    ┌──────────────┐
    │  SHOW LOADER │
    │  (Immediate  │
    │   feedback)  │
    └──────┬───────┘
           │
           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                        API 1: ZENDESK SDK LOGIN                           │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Purpose: Authenticate user with Zendesk SDK using JWT                    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  Zendesk.instance.loginUser(jwt = jwtToken)                         │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  On Success → Proceed to API 2                                            │
│  On Failure → Show "Couldn't login" error dialog                          │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
           │
           │ Success
           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     API 2: CREATE CONVERSATION                            │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Endpoint: POST {BASE_URL}/sc/v2/apps/{appId}/conversations               │
│  Auth: Basic Auth (appKey:appSecret)                                      │
│                                                                           │
│  Purpose: Create a new Sunshine Conversation with metadata                │
│                                                                           │
│  Request includes:                                                        │
│  • type: "personal"                                                       │
│  • displayName: subject (e.g., "Request Free Samples")                    │
│  • metadata: tags, NPI, email, name, source                               │
│  • participants: user's external ID (NPI)                                 │
│                                                                           │
│  Response returns: conversation.id                                        │
│                                                                           │
│  On Success → Store conversationId, proceed to API 3                      │
│  On Failure → Show error dialog                                           │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
           │
           │ Success (got conversationId)
           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                       API 3: SEND MESSAGE                                 │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Endpoint: POST {BASE_URL}/sc/v2/apps/{appId}/conversations/{id}/messages │
│  Auth: Basic Auth (appKey:appSecret)                                      │
│                                                                           │
│  Purpose: Post the user's request message to the conversation             │
│                                                                           │
│  Request includes:                                                        │
│  • author: { type: "user", userExternalId: NPI }                          │
│  • content: { type: "text", text: user's message }                        │
│                                                                           │
│  On Success → Proceed to API 4                                            │
│  On Failure → Show "Message sending failed" error dialog                  │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
           │
           │ Success (message sent)
           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      API 4: FETCH TICKET (POLLING)                        │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Endpoint: GET {BASE_URL}/api/v2/search.json?query=type:ticket tags:{tag} │
│  Auth: Basic Auth (agentEmail/token:apiToken)                             │
│                                                                           │
│  Purpose: Find the ticket that Zendesk auto-created from our conversation │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                         POLLING LOGIC                               │  │
│  │                                                                     │  │
│  │   ┌─────────┐     No ticket      ┌─────────┐                       │  │
│  │   │  POLL   │────────found──────▶│  WAIT   │                       │  │
│  │   │   API   │                    │  500ms  │                       │  │
│  │   └────┬────┘                    └────┬────┘                       │  │
│  │        │                              │                            │  │
│  │        │◀─────────────────────────────┘                            │  │
│  │        │                                                           │  │
│  │        │ Ticket found                                              │  │
│  │        ▼                                                           │  │
│  │   ┌─────────┐                                                      │  │
│  │   │ SUCCESS │                                                      │  │
│  │   └─────────┘                                                      │  │
│  │                                                                     │  │
│  │   Timeout: 30 seconds (if ticket not found → show error)           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  Response returns: results[0].id (ticketId)                               │
│                                                                           │
│  On Success → Store ticketId, proceed to API 5                            │
│  On Timeout → Show "Ticket creation timed out" error dialog               │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
           │
           │ Success (got ticketId)
           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      API 5: SET REQUESTER                                 │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Endpoint: PUT {BASE_URL}/api/v2/tickets/{ticketId}.json                  │
│  Auth: Basic Auth (agentEmail/token:apiToken)                             │
│                                                                           │
│  Purpose: Update ticket with prescriber's contact information             │
│                                                                           │
│  Request includes:                                                        │
│  • ticket.subject: request type                                           │
│  • ticket.requester: { name, external_id (NPI), phone, email }            │
│  • ticket.custom_fields: subject, NPI, conversationId                     │
│                                                                           │
│  On Success → Hide loader, open Zendesk Messaging UI                      │
│  On Failure → Show "Failed to set requester" error dialog                 │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
           │
           │ Success
           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     OPEN ZENDESK MESSAGING                                │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Action: Open Zendesk Messaging SDK with Most Recent Active Conversation  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  MessagingScreen.MostRecentActiveConversation(                      │  │
│  │      onExit = MessagingScreen.ExitAction.Close                      │  │
│  │  )                                                                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  WHY MostRecentActiveConversation (not Conversation by ID):               │
│  • Newly created conversation is the most recent one                      │
│  • Avoids sync issues where SDK can't find conversation by ID immediately │
│  • More reliable than waiting for conversation ID to be indexed           │
│                                                                           │
│  User can now:                                                            │
│  • See their submitted request                                            │
│  • Continue the conversation                                              │
│  • Receive replies from support team                                      │
│  • Get push notifications for new messages                                │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### API Flow Summary Table

| Step | API | Method | Purpose | Depends On |
|------|-----|--------|---------|------------|
| 1 | Zendesk SDK Login | SDK Call | Authenticate user | Valid JWT |
| 2 | Create Conversation | POST | Create new conversation | Step 1 success |
| 3 | Send Message | POST | Post user's request | Step 2 success (conversationId) |
| 4 | Fetch Ticket | GET (polling) | Find auto-created ticket | Step 3 success |
| 5 | Set Requester | PUT | Add prescriber info to ticket | Step 4 success (ticketId) |

---

## API Retry System

The Concierge feature implements a robust retry system that allows users to retry from any failed step without losing progress.

### Retry Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                         API RETRY STATE MACHINE                           │
└───────────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────┐
    │                     ConciergeApiTrigger Enum                        │
    │                                                                     │
    │   None ─────▶ CreateConversation ─────▶ SendMessage                 │
    │                      │                       │                      │
    │                      │                       ▼                      │
    │                      │               FetchTicket ─────▶ SetRequester│
    │                      │                       │                      │
    │                      ▼                       ▼                      │
    │              [On Failure]              [On Failure]                 │
    │                   │                         │                       │
    │                   └─────────┬───────────────┘                       │
    │                             ▼                                       │
    │                    Show Retry Dialog                                │
    │                             │                                       │
    │               ┌─────────────┴─────────────┐                        │
    │               │                           │                        │
    │               ▼                           ▼                        │
    │         [User taps OK]          [User taps Retry]                  │
    │               │                           │                        │
    │               ▼                           ▼                        │
    │         Reset State              Resume from failed step           │
    │         (start fresh)            (preserved data used)             │
    └─────────────────────────────────────────────────────────────────────┘
```

### Preserved Data on Failure

When an API step fails, the following data is preserved for retry:

| Data | Preserved For |
|------|---------------|
| `conversationId` | Steps 3, 4, 5 (if Step 2 succeeded) |
| `ticketId` | Step 5 (if Step 4 succeeded) |
| `convoTag` | All steps (unique identifier for ticket search) |
| `message` | All steps (user's request text) |
| `subject` | All steps (request type) |
| User profile data | All steps (NPI, email, name, phone) |

### Retry Dialog

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          API RETRY DIALOG                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│    ┌───────────────────────────────────────────────────────────────┐   │
│    │                                                               │   │
│    │                     ⚠️ {Step Name} Failed                     │   │
│    │                                                               │   │
│    │              {Error Message Description}                      │   │
│    │                                                               │   │
│    │    ┌─────────────┐              ┌─────────────┐              │   │
│    │    │     OK      │              │    Retry    │              │   │
│    │    └─────────────┘              └─────────────┘              │   │
│    │                                                               │   │
│    └───────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  OK Button:                                                             │
│  • Dismisses dialog                                                     │
│  • Calls resetSubmissionState()                                         │
│  • Clears all preserved data                                            │
│  • User returns to form with original input                             │
│                                                                         │
│  Retry Button:                                                          │
│  • Dismisses dialog                                                     │
│  • Calls retryCurrentStep()                                             │
│  • Uses preserved data (conversationId, ticketId, etc.)                 │
│  • Resumes from exact failed step                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Retry Behavior

| Step Failed | What's Preserved | Retry Starts From |
|-------------|------------------|-------------------|
| Create Conversation | Form data only | Step 2 (fresh conversation) |
| Send Message | conversationId | Step 3 (same conversation) |
| Fetch Ticket | conversationId | Step 4 (resume polling) |
| Set Requester | conversationId, ticketId | Step 5 (same ticket) |

### Implementation

```
private var apiTrigger: ConciergeApiTrigger = ConciergeApiTrigger.None

private fun showApiFailureDialog(
    stepName: String, 
    errorMsg: String, 
    failedTrigger: ConciergeApiTrigger
) {
    apiTrigger = failedTrigger
    _uiState.update {
        it.copy(
            showApiRetryDialog = true,
            apiRetryStepName = stepName,
            apiRetryErrorMessage = errorMsg,
            showLoader = false
        )
    }
}

fun retryCurrentStep() {
    _uiState.update { it.copy(showApiRetryDialog = false, showLoader = true) }
    processApiTrigger()
}

fun resetSubmissionState() {
    apiTrigger = ConciergeApiTrigger.None
    conversationId = null
    ticketId = null
}
```

### Retry in Request List & Archived Screens

Opening a conversation from the request list also has retry functionality:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 OPEN CONVERSATION RETRY FLOW                            │
└─────────────────────────────────────────────────────────────────────────┘

    User taps request card
            │
            ▼
    Step 1: Fetch conversation ID from ticket custom field
            │
            ├──── Success ────▶ Step 2: Login to Zendesk SDK
            │                          │
            │                          ├──── Success ────▶ Open Messaging
            │                          │
            │                          └──── Failure ────▶ Show Retry Dialog
            │
            └──── Failure ────▶ Show Retry Dialog
                                       │
                               ┌───────┴───────┐
                               │               │
                               ▼               ▼
                        [User taps OK]  [User taps Retry]
                               │               │
                               ▼               ▼
                        Dismiss dialog   Resume from failed step
                                        (uses cached conversationId
                                         if available)
```

### ConversationStep Enum

```
enum class ConversationStep {
    IDLE,
    FETCH_CONVERSATION,
    ZENDESK_LOGIN,
    OPEN_MESSAGING
}
```

---

## Zendesk Logout

When a user signs out of the app, they are also logged out of the Zendesk SDK and the cached JWT token is cleared.

### Logout Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          ZENDESK LOGOUT FLOW                            │
└─────────────────────────────────────────────────────────────────────────┘

    USER TAPS SIGN OUT
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      STEP 1: CLEAR JWT TOKEN                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  sharedViewModel.clearJwtToken()                                        │
│                                                                         │
│  Purpose: Remove cached JWT so next user gets fresh token               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      STEP 2: ZENDESK SDK LOGOUT                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Zendesk.instance.logoutUser()                                          │
│                                                                         │
│  On Success:                                                            │
│  • Reset login state flag                                               │
│  • Clear Zendesk session                                                │
│                                                                         │
│  On Failure:                                                            │
│  • Log error for debugging                                              │
│  • Reset login state flag anyway                                        │
│                                                                         │
│  Note: If Zendesk SDK is not initialized, logout is skipped             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
           │
           ▼
    PROCEED WITH APP SIGN OUT
```

### When Logout is Called

| Event | JWT Cleared | Zendesk Logout | Session Cleared |
|-------|-------------|----------------|-----------------|
| Sign Out (Remember Me = false) | ✅ | ✅ | ✅ |
| Sign Out (Remember Me = true) | ✅ | ✅ | ❌ (credentials kept) |
| Deactivate Account | ✅ | ✅ | ✅ |

---

## Screens & Components

### 1. Concierge Screen

**Purpose:** Main request submission interface

**Features:**
- Three mutually exclusive request types (radio-button behavior)
- Character-limited text input (200 chars max)
- Validation before submission

### 2. Request List Screen

**Purpose:** Display active requests only

**Active Request Status:**
- `new` - Fresh tickets awaiting initial response
- `open` - Tickets currently being worked on
- `pending` - Waiting for customer response

**Features:**
- Auto-refresh on screen resume (ON_RESUME lifecycle event)
- Tap card to open Zendesk messaging
- Dropdown menu for settings access (Contact Preferences, Archived Requests)
- Loading state management

### 3. Archived Request Screen

**Purpose:** Display all non-active requests

**Archived Request Status (any status NOT "new", "open", or "pending"):**
- `closed` - Request was resolved and closed
- `solved` - Request was completed successfully
- `hold` - Temporarily on hold
- Any other future status

**Features:**
- Auto-loads on screen mount
- Tap card to view conversation history in Zendesk messaging
- Loading and error states

### 4. Settings Screen

**Purpose:** Navigation hub for settings

**Features:**
- Link to Contact Preferences
- Link to Archived Requests

### 5. Contact Preferences Screen

**Purpose:** Set contact hour preferences

**Features:**
- Day-by-day time range selection
- "Do Not Contact" toggle per day
- Save to Zendesk user profile

---

## Ticket Status Filtering

The app categorizes Zendesk tickets into two groups based on their status:

### Active Requests (Request List Screen)

Tickets are considered **active** if their status is:

| Status | Description |
|--------|-------------|
| `new` | Fresh ticket awaiting initial response |
| `open` | Ticket currently being worked on |
| `pending` | Waiting for customer response |

### Archived Requests (Archived Screen)

Tickets are considered **archived** if their status is **anything other than "new", "open", or "pending"**:

| Status | Description |
|--------|-------------|
| `closed` | Request was resolved and closed |
| `solved` | Request was completed successfully |
| `hold` | Temporarily on hold |
| *any other* | Future statuses automatically go here |

### Implementation

The filtering logic is implemented in the ViewModel:

```
private fun ZendeskRequest.isActive(): Boolean {
    return status.equals("new", ignoreCase = true) ||
            status.equals("open", ignoreCase = true) ||
            status.equals("pending", ignoreCase = true)
}

private fun ZendeskRequest.isArchived(): Boolean {
    return !isActive()
}
```

The UI state provides computed properties for easy access:

```
data class RequestsListUiState(...) {
    val activeRequests: List<ZendeskRequest>
        get() = allRequests.filterNot { it.isArchived() }

    val archivedRequests: List<ZendeskRequest>
        get() = allRequests.filter { it.isArchived() }
}
```

---

## API Endpoints

### Base URL

```
{BASE_URL} = Your Zendesk instance URL (e.g., https://yourcompany.zendesk.com)
```

### 1. Create Conversation (Zendesk Sunshine)
```
POST {BASE_URL}/sc/v2/apps/{appId}/conversations
Authorization: Basic {appKey}:{appSecret}
Content-Type: application/json

Request Body:
{
  "type": "personal",
  "displayName": "{subject}",
  "metadata": {
    "zen:ticket:tags": "mobile,android,concierge,prescriber,from_mobile,{uniqueTag}",
    "provider_npi": "{npi}",
    "provider_email": "{email}",
    "provider_name": "{name}",
    "source": "android_app",
    "client_conversation_tag": "{uniqueTag}"
  },
  "participants": [
    { "userExternalId": "{npi}" }
  ]
}

Response:
{
  "conversation": {
    "id": "conversation_id_here"
  }
}
```

### 2. Send Message (Zendesk Sunshine)
```
POST {BASE_URL}/sc/v2/apps/{appId}/conversations/{conversationId}/messages
Authorization: Basic {appKey}:{appSecret}
Content-Type: application/json

Request Body:
{
  "author": {
    "type": "user",
    "userExternalId": "{npi}"
  },
  "content": {
    "type": "text",
    "text": "{message}"
  }
}

Response:
{
  "message": {
    "id": "message_id_here"
  }
}
```

### 3. Search Ticket by Tag
```
GET {BASE_URL}/api/v2/search.json?query=type:ticket tags:{uniqueTag}
Authorization: Basic {agentEmail}/token:{apiToken}

Response:
{
  "results": [
    { 
      "id": 12345, 
      "subject": "Request Free Samples",
      "status": "new"
    }
  ]
}
```

### 4. Update Ticket (Set Requester)
```
PUT {BASE_URL}/api/v2/tickets/{ticketId}.json
Authorization: Basic {agentEmail}/token:{apiToken}
Content-Type: application/json

Request Body:
{
  "ticket": {
    "subject": "{subject}",
    "requester": {
      "name": "{name}",
      "external_id": "{npi}",
      "phone": "{phone}",
      "email": "{email}"
    },
    "custom_fields": [
      { "id": {customFieldId1}, "value": "{subject}" },
      { "id": {customFieldId2}, "value": "{npi}" },
      { "id": {customFieldId3}, "value": "{conversationId}" }
    ]
  }
}
```

### 5. Search User by Email (for Requests List)
```
GET {BASE_URL}/api/v2/users/search.json?query=email:{userEmail}
Authorization: Basic {agentEmail}/token:{apiToken}

Response:
{
  "users": [
    { "id": 12345, "email": "user@example.com" }
  ]
}
```

### 6. Get User's Tickets
```
GET {BASE_URL}/api/v2/users/{userId}/tickets/requested.json
Authorization: Basic {agentEmail}/token:{apiToken}

Response:
{
  "tickets": [
    { "id": 123, "subject": "...", "status": "open" },
    { "id": 124, "subject": "...", "status": "solved" }
  ]
}
```

### 7. Update User Contact Preferences
```
PUT {BASE_URL}/api/v2/users/{userId}.json
Authorization: Basic {agentEmail}/token:{apiToken}
Content-Type: application/json

Request Body:
{
  "user": {
    "user_fields": {
      "monday_contact_hours": "9:00 AM - 5:00 PM",
      "tuesday_contact_hours": "No",
      "wednesday_contact_hours": "10:00 AM - 6:00 PM"
    }
  }
}
```

---

## Error Handling

### Validation Errors

| Condition | Error Message | UI Action |
|-----------|---------------|-----------|
| No request type selected | "Please enter a request." | Show error dialog |
| Empty message content | "Please enter a request." | Show error dialog, focus input |
| Missing NPI | "Your NPI number is missing..." | Show hyperlink dialog (edit profile) |
| Missing phone | "Your phone number is missing..." | Show hyperlink dialog |
| Missing both | "Your phone number and NPI..." | Show hyperlink dialog |

### API Errors

| HTTP Code | Meaning | Handling |
|-----------|---------|----------|
| -1 | Network failure | "Network error" dialog |
| 11 | Connectivity issue | "Internet connectivity or server issue" |
| 200-299 | Success | Proceed to next step |
| 400-499 | Client error | "Something went wrong" |
| 500+ | Server error | "Something went wrong" |
| Timeout | Ticket not found in 30s | "Ticket creation timed out" |

### JWT Token Error with Retry

When submitting a Concierge request, if the JWT token is null, empty, or expired, a special retry dialog is shown with two options:

| Button | Action |
|--------|--------|
| **OK** | Dismiss dialog, return to form |
| **Retry** | Regenerate JWT token, re-login to Zendesk, and proceed with submission |

**Retry Flow:**
1. User clicks "Retry"
2. System generates a new JWT token using user's NPI, name, and email
3. System attempts Zendesk SDK login with the new token
4. On success: Proceeds with the request submission
5. On failure: Shows the retry dialog again

This handles cases where:
- JWT token was never generated
- JWT token has expired (6-hour expiry)
- JWT token was cleared from memory

### Error Flow

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          ERROR HANDLING FLOW                              │
└───────────────────────────────────────────────────────────────────────────┘

   Any API Call
        │
        ├──── Success ────▶ Continue to next step
        │
        └──── Failure ────▶ ┌─────────────────┐
                           │  Hide Loader     │
                           │  Show Error      │
                           │  Dialog          │
                           │                  │
                           │  User taps OK    │
                           │       │          │
                           │       ▼          │
                           │  Return to form  │
                           │  (data preserved)│
                           └─────────────────┘
```

---

## Security Considerations

1. **JWT Authentication:** Zendesk SDK login uses JWT tokens with 6-hour expiry
2. **Single Source of Truth:** JWT is managed centrally via `SharedViewModel.getOrCreateJwtToken()`
3. **JWT Cleanup:** JWT is cleared on logout/deactivation via `clearJwtToken()`
4. **Basic Auth:** Sunshine Conversations API uses app key/secret (Base64 encoded)
5. **Token Storage:** Access tokens should be stored securely (encrypted storage recommended)
6. **NPI as External ID:** User's NPI is used as the external identifier in Zendesk

---

