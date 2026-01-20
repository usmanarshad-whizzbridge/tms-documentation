# LTI Restoration Methodology - AWS Integration

**Document Version:** 1.0
**Date:** January 20, 2026
**Prepared for:** AWS

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Overview](#overview)
3. [Architecture Components](#architecture-components)
4. [User Journey - Sequence Diagram](#user-journey-sequence-diagram)
5. [Detailed Flow](#detailed-flow)
6. [API Endpoints](#api-endpoints)
7. [Authentication & Security](#authentication--security)
8. [Data Model](#data-model)
9. [Technical Implementation](#technical-implementation)

---

## Executive Summary

The LTI Restoration Methodology describes how the TMS (Training Management System) platform integrates with AWS using LTI 1.3 standards to launch courses and restore (fetch) learner grades and progress data. This bidirectional integration enables seamless course delivery and comprehensive learning analytics.

---

## Overview

### Integration Model: Push vs Pull

**IMPORTANT**: The LTI restoration methodology uses a **push model** for grades:

| Service | Direction | Model | Purpose |
|---------|-----------|-------|---------|
| **AGS (Grades)** | AWS → TMS | **PUSH** | AWS sends scores TO TMS when learners complete activities |
| **NRPS (Roster)** | AWS ← TMS | **PULL** | AWS fetches enrolled learners FROM TMS (optional) |

The integration consists of two primary phases:

### Phase 1: Launch Flow
- Learner initiates course from TMS platform
- Secure OIDC authentication handshake with AWS
- Learner accesses AWS course content
- Auto-enrollment in external course

### Phase 2: Restoration Flow (Data Synchronization)

**AWS → TMS (Push Model - Primary)**
- **AGS (Assignment and Grade Services)**: AWS **pushes** grades and scores TO TMS when learners complete activities
- TMS acts as **AGS Provider** (receives grades)
- Automatic update of learner enrollment records (`self_courses`)

**AWS ← TMS (Pull Model - Optional)**
- **NRPS (Names and Role Provisioning Services)**: AWS **fetches** learner roster FROM TMS
- TMS acts as **NRPS Provider** (provides roster data)
- AWS can query enrolled members and their basic progress data

### Data Flow Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    LTI Integration Flow                     │
└─────────────────────────────────────────────────────────────┘

1. LAUNCH PHASE (User Initiates)
   Learner → TMS → AWS

2. LEARNING PHASE
   Learner completes activities in AWS

3. GRADE RESTORATION (Automatic - Push Model)
   AWS → TMS (POST /api/v1/lti/ags/lineitems/{id}/scores)
   ├─ AWS creates line items
   ├─ AWS posts scores when learner completes
   └─ TMS auto-updates self_courses table

4. ROSTER SYNC (Optional - Pull Model)
   AWS ← TMS (GET /api/v1/lti/nrps/contexts/{id}/memberships)
   └─ AWS fetches enrolled learner list from TMS
```

---

## Architecture Components

### Core Components

| Component | Location | Purpose |
|-----------|----------|---------|
| Launch Controller | `app/controllers/lti/launches_controller.rb` | Handles LTI launch initiation and authorization |
| Grades Fetcher | `app/services/lti/grades_fetcher.rb` | AGS service for retrieving grades from AWS |
| Progress Fetcher | `app/services/lti/progress_fetcher.rb` | NRPS service for retrieving progress from AWS |
| Grades Controller | `app/controllers/api/v1/lti/grades_controller.rb` | API endpoints for grade restoration |
| Progress Controller | `app/controllers/api/v1/lti/progress_controller.rb` | API endpoints for progress restoration |
| ID Token Builder | `app/services/lti/id_token_builder.rb` | Constructs JWT tokens for AWS |
| Token Fetcher | `app/services/lti/token_fetcher.rb` | OAuth 2.0 token management |

### Supporting Models

- **LtiTool**: AWS tool configuration
- **LtiDeployment**: Deployment instances for different contexts
- **LtiSession**: Temporary session storage for launch flow
- **LtiExternalCourseConfig**: Course-specific LTI configuration
- **SelfCourse**: Learner enrollment records
- **LtiLineItem**: Cached assignment/assessment data
- **LtiResult**: Cached grade results

---

## User Journey - Sequence Diagram

```
┌─────────┐      ┌──────────┐      ┌─────────┐      ┌──────────────┐
│ Learner │      │React App │      │TMS API  │      │AWS│
└────┬────┘      └────┬─────┘      └────┬────┘      └──────┬───────┘
     │                │                  │                  │
     │ 1. Click Launch Course            │                  │
     ├───────────────>│                  │                  │
     │                │                  │                  │
     │                │ 2. POST /lti/launch/:deployment_id  │
     │                │    (user_id, context_id,            │
     │                │     resource_link_id)               │
     │                ├─────────────────>│                  │
     │                │                  │                  │
     │                │                  │ 3. Create LtiSession
     │                │                  │    (state, nonce, session_key)
     │                │                  │                  │
     │                │                  │ 4. Auto-enroll learner
     │                │                  │    to external course
     │                │                  │    (if not enrolled)
     │                │                  │                  │
     │                │ 5. Redirect to AWS Login URL        │
     │                │    (with state, nonce, login_hint)  │
     │                │<─────────────────┤                  │
     │                │                                     │
     │ 6. Browser redirect to AWS                           │
     ├────────────────┼─────────────────┼─────────────────>│
     │                │                  │                  │
     │                │                  │ 7. AWS OIDC Auth Callback
     │                │                  │    GET /lti/authorize
     │                │                  │<─────────────────┤
     │                │                  │                  │
     │                │                  │ 8. Validate session
     │                │                  │    Destroy LtiSession
     │                │                  │                  │
     │                │                  │ 9. Build JWT ID Token
     │                │                  │    (user info, context,
     │                │                  │     AGS/NRPS endpoints)
     │                │                  │                  │
     │                │                  │ 10. POST form with id_token
     │                │                  ├─────────────────>│
     │                │                  │                  │
     │ 11. AWS course loads in new tab                │
     │<──────────────────────────────────┼──────────────────┤
     │                │                  │                  │
     │ 12. Learner completes activities  │                  │
     ├──────────────────────────────────────────────────────>│
     │                │                  │                  │
     │                │                  │                  │
╔════════════════════════════════════════════════════════════════╗
║  RESTORATION PHASE: AWS Pushes Grades to TMS                 ║
╚════════════════════════════════════════════════════════════════╝
     │                │                  │                  │
     │                │                  │ 13. AWS creates line item (assignment)
     │                │                  │     Request OAuth token
     │                │                  │     POST /lti/token
     │                │                  │<─────────────────┤
     │                │                  │                  │
     │                │                  │ 14. Return access_token
     │                │                  ├─────────────────>│
     │                │                  │                  │
     │                │                  │ 15. Create line item on TMS
     │                │                  │     POST /api/v1/lti/ags/contexts/{context_id}/lineitems
     │                │                  │     Authorization: Bearer {token}
     │                │                  │<─────────────────┤
     │                │                  │                  │
     │                │                  │ 16. Line item created
     │                │                  ├─────────────────>│
     │                │                  │                  │
     │ 17. Learner completes activity/assessment             │
     ├──────────────────────────────────────────────────────>│
     │                │                  │                  │
     │                │                  │ 18. AWS calculates score
     │                │                  │     Request OAuth token
     │                │                  │     POST /lti/token
     │                │                  │<─────────────────┤
     │                │                  │                  │
     │                │                  │ 19. Return access_token
     │                │                  ├─────────────────>│
     │                │                  │                  │
     │                │                  │ 20. Push score to TMS
     │                │                  │     POST /api/v1/lti/ags/lineitems/{id}/scores
     │                │                  │     Authorization: Bearer {token}
     │                │                  │     Body: {
     │                │                  │       userId: "user-xxx",
     │                │                  │       scoreGiven: 85,
     │                │                  │       scoreMaximum: 100,
     │                │                  │       activityProgress: "Completed",
     │                │                  │       gradingProgress: "FullyGraded"
     │                │                  │     }
     │                │                  │<─────────────────┤
     │                │                  │                  │
     │                │                  │ 21. Store score & update SelfCourse
     │                │                  │     - Save to lti_results
     │                │                  │     - Update self_course.obtained_marks
     │                │                  │     - Update self_course.progress
     │                │                  │     - Update self_course.evaluation
     │                │                  │     - Update self_course.status
     │                │                  │                  │
     │                │                  │ 22. Score accepted
     │                │                  ├─────────────────>│
     │                │                  │                  │
     │ 23. Learner sees updated grade in TMS                 │
     │<───────────────┤                  │                  │
     │                │                  │                  │
```

---

## Detailed Flow

### Launch Phase (Steps 1-12)

#### Step 1-2: Launch Initiation
- User clicks "Launch Course" in the React application
- React app sends POST request to `/lti/launch/:deployment_id` with:
   - `user_id`: Current user identifier
   - `context_id`: Course ID
   - `resource_link_id`: Specific resource within course

#### Step 3: Session Creation
TMS creates a secure temporary session with:
- **State**: CSRF protection token (32-byte random)
- **Nonce**: Replay attack prevention (32-byte random)
- **Session Key**: Session identifier (32-byte random)
- **Expiration**: 10 minutes from creation

Implementation location: `app/controllers/lti/launches_controller.rb:54-73`

#### Step 4: Auto-Enrollment
If the course is an external LTI course and the learner is not enrolled:
- Creates `SelfCourse` enrollment record
- Links learner to the course
- Enables progress tracking

Implementation location: `app/controllers/lti/launches_controller.rb:124-166`

#### Step 5-6: OIDC Redirect
TMS constructs and redirects to AWS's login URL with parameters:
- `iss`: Platform issuer identifier
- `login_hint`: Session key for restoration
- `lti_message_hint`: Message type indicator
- `client_id`: AWS tool client ID
- `target_link_uri`: Final destination URL
- `state`: CSRF token
- `nonce`: Anti-replay token

#### Step 7-9: Authorization Callback
AWS redirects back to TMS `/lti/authorize` endpoint:
- TMS validates the session using `login_hint`
- Retrieves stored session data
- Destroys the temporary session (one-time use)
- Builds JWT ID token with user claims and service endpoints

#### Step 10-11: Token Submission
TMS generates an auto-submitting HTML form that POSTs to AWS with:
- `id_token`: Signed JWT containing user info and service endpoints
- `state`: Original state parameter for validation

The JWT token includes:
- User identification (sub, name, email)
- Context information (course ID)
- AGS endpoint URLs (for grade reporting)
- NRPS endpoint URLs (for roster access)
- Custom parameters (course-specific data)

Implementation location: `app/services/lti/id_token_builder.rb`

#### Step 12: Course Access
AWS validates the ID token and loads the course for the learner.

---

### Restoration Phase - Grades (Steps 13-23)

#### Purpose
AWS **pushes** assignment/assessment grades to TMS automatically when learners complete activities. TMS acts as an AGS Provider (receives grades), not a consumer (fetches grades).

#### Step 13-14: AWS Requests OAuth Token
When AWS needs to create a line item or post a score, it requests an access token from TMS:
```
POST /lti/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion={signed_jwt}
&scope=https://purl.imsglobal.org/spec/lti-ags/scope/lineitem https://purl.imsglobal.org/spec/lti-ags/scope/score
```

TMS validates the JWT and returns an access token.

Implementation location: `app/controllers/lti/token_controller.rb`

#### Step 15-16: AWS Creates Line Item
AWS creates an assignment (line item) on TMS:
```
POST /api/v1/lti/ags/contexts/{context_id}/lineitems
Authorization: Bearer {access_token}
Content-Type: application/vnd.ims.lis.v2.lineitem+json

{
  "label": "Module 1 Assessment",
  "scoreMaximum": 100,
  "resourceLinkId": "resource-123",
  "tag": "assessment"
}
```

TMS stores the line item in `lti_line_items` table.

Implementation location: `app/controllers/api/v1/lti/ags/lineitems_controller.rb:28-42`

#### Step 17: Learner Completes Activity
Learner completes an assessment or activity in AWS.

#### Step 18-19: AWS Requests Token for Score Posting
AWS requests another access token with score scope.

#### Step 20-21: AWS Pushes Score to TMS
AWS posts the learner's score to TMS:
```
POST /api/v1/lti/ags/lineitems/{lineitem_id}/scores
Authorization: Bearer {access_token}
Content-Type: application/vnd.ims.lis.v1.score+json

{
  "userId": "user-4750fd22-7b2b-4bd7-9f00-7df80f3b93f1",
  "scoreGiven": 85,
  "scoreMaximum": 100,
  "comment": "Excellent work!",
  "activityProgress": "Completed",
  "gradingProgress": "FullyGraded",
  "timestamp": "2026-01-20T10:30:00Z"
}
```

TMS:
1. Validates the token and scope
2. Strips "user-" prefix from userId to get internal user ID
3. Finds or creates `LtiResult` record
4. Saves the score
5. **Automatically updates the `SelfCourse`** enrollment:
   - `obtained_marks` = percentage score (e.g., 85%)
   - `progress` = 100 if activityProgress is "Completed"
   - `evaluation` = "passed" or "failed" based on passing_marks
   - `status` = "completed" if activity is completed

Implementation location: `app/controllers/api/v1/lti/ags/scores_controller.rb`

#### Step 22: TMS Confirms Score Receipt
TMS responds with success:
```json
{
  "resultUrl": "https://tms.example.com/api/v1/lti/ags/lineitems/123/results"
}
```

#### Step 23: Grade Available in TMS
The learner's grade is now available in TMS and reflected in their course enrollment (`self_courses` table).

---

### NRPS - AWS Fetches Member Roster from TMS (Optional)

#### Purpose
AWS can **fetch** (pull) the learner roster from TMS to understand who is enrolled in a course. This allows AWS to display enrolled learners and potentially sync rosters.

**Note**: This is optional and depends on whether AWS requests roster information.

#### How It Works

When AWS needs roster information, it:

1. **Requests OAuth Token** from TMS:
```
POST /lti/token
grant_type=client_credentials
&scope=https://purl.imsglobal.org/spec/lti-nrps/scope/contextmembership.readonly
```

2. **Fetches Memberships** from TMS:
```
GET /api/v1/lti/nrps/contexts/{context_id}/memberships
Authorization: Bearer {access_token}
Accept: application/vnd.ims.lti-nrps.v2.membershipcontainer+json
```

3. **TMS Responds** with roster data:
```json
{
  "id": "https://tms.example.com/api/v1/lti/nrps/contexts/123/memberships",
  "context": {
    "id": "123",
    "label": "Introduction to AI",
    "title": "AI Fundamentals",
    "type": ["http://purl.imsglobal.org/vocab/lis/v2/course#CourseOffering"]
  },
  "members": [
    {
      "status": "Active",
      "name": "John Doe",
      "email": "john@example.com",
      "user_id": "456",
      "roles": ["http://purl.imsglobal.org/vocab/lis/v2/membership#Learner"],
      "message": {
        "obtained_marks": 85,
        "evaluation": "passed"
      }
    }
  ]
}
```

TMS fetches enrolled learners from `event_learners` table where the event uses this course.

Implementation location: `app/controllers/api/v1/lti/nrps/memberships_controller.rb`

**Direction**: AWS → TMS (AWS pulls roster FROM TMS)

---

## API Endpoints

### Category Overview

The TMS API provides three categories of LTI endpoints:

1. **Launch Endpoints**: Initiate LTI course launches
2. **AGS Provider Endpoints**: Receive grades FROM AWS (AWS calls these)
3. **NRPS Provider Endpoints**: Provide roster TO AWS (AWS calls these)
4. **Consumer Endpoints**: Fetch data from AWS (TMS admins call these - optional/testing)

---

### Launch Endpoints

#### Start Launch
```
POST /lti/launch/:deployment_id
```

**Parameters:**
- `deployment_id` (path): LTI deployment identifier
- `user_id` (body): Current user ID
- `context_id` (body): Course/context ID
- `resource_link_id` (body, optional): Specific resource
- Custom parameters with `custom_` prefix

**Response:** Redirects to AWS login URL

---

#### Authorization Callback
```
GET/POST /lti/authorize
```

**Parameters:**
- `login_hint`: Session key from launch
- `state`: CSRF protection token
- `nonce`: Anti-replay token

**Response:** Auto-submitting HTML form with JWT

---

### AGS Provider Endpoints (AWS Calls These)

These endpoints are called BY AWS to push grades TO TMS. They require OAuth authentication with Bearer tokens issued by TMS.

#### Create Line Item
```
POST /api/v1/lti/ags/contexts/:context_id/lineitems
Authorization: Bearer {access_token}
Content-Type: application/vnd.ims.lis.v2.lineitem+json
```

**Called by**: AWS
**Purpose**: Create an assignment/assessment (line item) on TMS

**Request Body:**
```json
{
  "label": "Module 1 Assessment",
  "scoreMaximum": 100,
  "resourceLinkId": "resource-123",
  "resourceId": "assessment-456",
  "tag": "graded",
  "startDateTime": "2026-01-20T00:00:00Z",
  "endDateTime": "2026-02-20T23:59:59Z"
}
```

**Response:**
```json
{
  "id": "https://tms.example.com/api/v1/lti/ags/lineitems/789",
  "label": "Module 1 Assessment",
  "scoreMaximum": 100,
  ...
}
```

Implementation: `app/controllers/api/v1/lti/ags/lineitems_controller.rb:28-42`

---

#### Post Score to TMS
```
POST /api/v1/lti/ags/lineitems/:lineitem_id/scores
Authorization: Bearer {access_token}
Content-Type: application/vnd.ims.lis.v1.score+json
```

**Called by**: AWS
**Purpose**: Push a learner's score to TMS after activity completion

**Request Body:**
```json
{
  "userId": "user-4750fd22-7b2b-4bd7-9f00-7df80f3b93f1",
  "scoreGiven": 85,
  "scoreMaximum": 100,
  "comment": "Excellent work!",
  "activityProgress": "Completed",
  "gradingProgress": "FullyGraded",
  "timestamp": "2026-01-20T10:30:00Z"
}
```

**What TMS Does:**
1. Strips "user-" prefix from userId
2. Stores score in `lti_results` table
3. **Automatically updates `self_courses`**:
   - `obtained_marks` = percentage score
   - `progress` = 100 if Completed
   - `evaluation` = passed/failed
   - `status` = completed

**Response:**
```json
{
  "resultUrl": "https://tms.example.com/api/v1/lti/ags/lineitems/789/results"
}
```

Implementation: `app/controllers/api/v1/lti/ags/scores_controller.rb`

---

#### Get Line Items (AWS reads)
```
GET /api/v1/lti/ags/contexts/:context_id/lineitems
Authorization: Bearer {access_token}
Accept: application/vnd.ims.lis.v2.lineitemcontainer+json
```

**Called by**: AWS
**Purpose**: AWS fetches list of line items it previously created

**Query Parameters:**
- `resource_id` (optional): Filter by resource ID
- `resource_link_id` (optional): Filter by resource link ID
- `tag` (optional): Filter by tag
- `limit` (optional): Pagination limit (max 100)

Implementation: `app/controllers/api/v1/lti/ags/lineitems_controller.rb:7-26`

---

#### Get Results (AWS reads)
```
GET /api/v1/lti/ags/lineitems/:lineitem_id/results
Authorization: Bearer {access_token}
Accept: application/vnd.ims.lis.v2.resultcontainer+json
```

**Called by**: AWS
**Purpose**: AWS fetches results/scores it previously posted

Implementation: `app/controllers/api/v1/lti/ags/results_controller.rb`

---

### AGS Consumer Endpoints (TMS Admins Call - Optional/Testing)

These endpoints allow TMS administrators to manually fetch grades FROM AWS. These are optional and primarily for testing.

#### Fetch All Grades from AWS
```
GET /api/v1/lti/tools/:tool_id/grades?context_id=<course_id>
Authorization: Bearer {tms_auth_token}
```

**Called by**: TMS administrators
**Purpose**: Manually pull all grades from AWS (for testing/debugging)

Implementation: `app/controllers/api/v1/lti/grades_controller.rb`

---

#### Fetch Progress from AWS
```
GET /api/v1/lti/tools/:tool_id/progress/learners?context_id=<course_id>
Authorization: Bearer {tms_auth_token}
```

**Called by**: TMS administrators
**Purpose**: Manually pull learner progress from AWS (for testing/debugging)

Implementation: `app/controllers/api/v1/lti/progress_controller.rb`

---

### NRPS Provider Endpoints (AWS Calls These)

These endpoints are called BY AWS to fetch roster information FROM TMS.

#### Get Course Memberships
```
GET /api/v1/lti/nrps/contexts/:context_id/memberships
Authorization: Bearer {access_token}
Accept: application/vnd.ims.lti-nrps.v2.membershipcontainer+json
```

**Called by**: AWS
**Purpose**: AWS fetches the list of enrolled learners from TMS

**Query Parameters:**
- `role` (optional): Filter by role (e.g., Learner, Instructor)
- `limit` (optional): Pagination limit (max 100, default 50)

**Response:**
```json
{
  "id": "https://tms.example.com/api/v1/lti/nrps/contexts/123/memberships",
  "context": {
    "id": "123",
    "label": "intro-ai",
    "title": "Introduction to AI",
    "type": ["http://purl.imsglobal.org/vocab/lis/v2/course#CourseOffering"]
  },
  "members": [
    {
      "status": "Active",
      "name": "John Doe",
      "email": "john@example.com",
      "user_id": "456",
      "roles": ["http://purl.imsglobal.org/vocab/lis/v2/membership#Learner"],
      "message": {
        "obtained_marks": 85,
        "rating": 4,
        "evaluation": "passed"
      }
    }
  ]
}
```

**Data Source**: TMS fetches from `event_learners` table (learners enrolled in events that include this course)

Implementation: `app/controllers/api/v1/lti/nrps/memberships_controller.rb`

---

## Authentication & Security

### LTI 1.3 Security Features

#### 1. JWT ID Token
- **Algorithm**: RS256 (RSA Signature with SHA-256)
- **Key Management**: Platform maintains public/private key pair
- **Claims**: Standard LTI claims plus custom parameters
- **Expiration**: Short-lived (typically 1 hour)
- **Audience**: AWS client ID

File: `app/services/lti/id_token_builder.rb`

#### 2. Nonce Validation
- **Purpose**: Prevent replay attacks
- **Storage**: Database with expiration timestamp
- **Lifecycle**: Single use, 10-minute validity
- **Cleanup**: Automatic expiration

Table: `lti_nonces`

#### 3. State Parameter
- **Purpose**: CSRF protection
- **Generation**: Cryptographically secure random (32 bytes)
- **Validation**: Round-trip verification
- **Storage**: In LtiSession with session_key

#### 4. OAuth 2.0 Token Management
- **Grant Type**: Client Credentials
- **Token Caching**: In-memory with expiration tracking
- **Scopes**: Service-specific (AGS, NRPS)
- **Security**: Bearer token in Authorization header

File: `app/services/lti/token_fetcher.rb`

#### 5. Session Security
- **Storage**: Database-backed (not cookie-based)
- **Expiration**: 10 minutes
- **One-time Use**: Destroyed after authorization
- **Key Generation**: Secure random

Table: `lti_sessions`

### Security Headers

#### Referrer Policy
```
Referrer-Policy: no-referrer
```
Prevents browser from sending referrer header to avoid domain mismatch issues.

#### HTTPS Enforcement
All LTI endpoints require HTTPS in production.

---

## Data Model

### LTI Tool Configuration

#### lti_tools
Stores AWS tool registration.

```ruby
{
  id: integer,
  name: string,                    # "AWS"
  client_id: string,               # OAuth client ID
  platform_public_jwks_url: string, # Platform's JWKS endpoint
  login_url: string,               # AWS OIDC login URL
  launch_url: string,              # AWS launch endpoint
  public_jwk: text,                # AWS's public key (JWK format)
  active: boolean,                 # Enable/disable tool
  created_at: datetime,
  updated_at: datetime
}
```

File: `app/models/lti_tool.rb`

---

#### lti_deployments
Represents tool deployment instances.

```ruby
{
  id: integer,
  lti_tool_id: integer,            # Foreign key to lti_tools
  deployment_id: string,           # Unique deployment identifier
  name: string,                    # "Production Deployment"
  created_at: datetime,
  updated_at: datetime
}
```

File: `app/models/lti_deployment.rb`

---

### Launch Session Management

#### lti_sessions
Temporary storage for launch flow.

```ruby
{
  id: integer,
  session_key: string,             # Unique session identifier
  state: string,                   # CSRF token
  lti_deployment_id: integer,      # Foreign key
  user_id: integer,                # Foreign key to users
  context_id: string,              # Course ID
  resource_link_id: string,        # Resource identifier
  custom_params: text,             # JSON of custom parameters
  expires_at: datetime,            # 10 minutes from creation
  created_at: datetime,
  updated_at: datetime
}
```

File: `app/models/lti_session.rb`

---

#### lti_nonces
Anti-replay protection.

```ruby
{
  id: integer,
  value: string,                   # Unique nonce value
  expires_at: datetime,            # 10 minutes from creation
  created_at: datetime
}
```

File: `app/models/lti_nonce.rb`

---

### Course Configuration

#### lti_external_course_configs
Course-specific LTI settings.

```ruby
{
  id: integer,
  course_id: integer,              # Foreign key to courses
  lti_tool_id: integer,            # Foreign key to lti_tools
  provider: string,                # "AWS"
  external_course_id: string,      # AWS course UUID (exportUuid)
  external_course_url: string,     # Direct course URL
  context_id_override: string,     # Override default context_id
  resource_link_id_override: string, # Override default resource_link_id
  external_thumbnail_url: string,  # Course image
  external_duration_hours: decimal, # Course duration
  external_level: string,          # Difficulty level
  auto_sync_grades: boolean,       # Enable auto grade sync
  auto_sync_progress: boolean,     # Enable auto progress sync
  active: boolean,                 # Enable/disable config
  custom_params: jsonb,            # Additional custom parameters
  external_metadata: jsonb,        # Extra metadata from AWS
  created_at: datetime,
  updated_at: datetime
}
```

File: `app/models/lti_external_course_config.rb`

---

### Grade Storage (Cached)

#### lti_line_items
Cached assignment/assessment data from AWS.

```ruby
{
  id: integer,
  lti_tool_id: integer,            # Foreign key
  context_id: string,              # Course ID
  lineitem_url: string,            # AWS line item URL
  resource_link_id: string,        # Associated resource
  resource_id: string,             # AWS resource ID
  label: string,                   # Assignment name
  tag: string,                     # Classification tag
  score_maximum: decimal,          # Maximum score
  start_datetime: datetime,        # Assignment start
  end_datetime: datetime,          # Assignment end
  created_at: datetime,
  updated_at: datetime
}
```

File: `app/models/lti_line_item.rb`

---

#### lti_results
Cached grade results from AWS.

```ruby
{
  id: integer,
  lti_line_item_id: integer,       # Foreign key
  user_id: string,                 # Learner identifier
  result_score: decimal,           # Score achieved
  result_maximum: decimal,         # Maximum possible
  comment: text,                   # Feedback
  timestamp: datetime,             # Result timestamp
  created_at: datetime,
  updated_at: datetime
}
```

File: `app/models/lti_result.rb`

---

### Access Token Management

#### lti_access_tokens
OAuth token cache.

```ruby
{
  id: integer,
  lti_tool_id: integer,            # Foreign key
  access_token: text,              # Bearer token
  token_type: string,              # "Bearer"
  expires_at: datetime,            # Token expiration
  scope: string,                   # Requested scopes
  created_at: datetime,
  updated_at: datetime
}
```

File: `app/models/lti_access_token.rb`

---

## Technical Implementation

### Key Files and Their Roles

#### Launch Flow
1. **`app/controllers/lti/launches_controller.rb`**
   - Handles `/lti/launch/:deployment_id` (start)
   - Handles `/lti/authorize` (callback)
   - User resolution (JWT, current_user, fallback)
   - Session creation and validation
   - Auto-enrollment logic
   - OIDC URL construction

2. **`app/services/lti/id_token_builder.rb`**
   - Constructs JWT ID token
   - Includes user claims
   - Adds AGS/NRPS endpoint URLs
   - Signs token with platform private key
   - Custom parameter injection

3. **`lib/lti/key_provider.rb`**
   - Manages RSA key pair
   - Provides public JWKS endpoint
   - Signs JWT tokens

4. **`app/controllers/lti/jwks_controller.rb`**
   - Exposes public key at `/lti/jwks`
   - Used by AWS to validate signatures

5. **`app/controllers/lti/token_controller.rb`**
   - OAuth token endpoint `/lti/token`
   - Issues access tokens to AWS
   - Validates client credentials

---

#### AGS Provider Flow (Receive Grades from AWS)

1. **`app/controllers/api/v1/lti/ags/lineitems_controller.rb`**
   - Handles line item CRUD operations
   - AWS creates, reads, updates, deletes assignments
   - Stores in `lti_line_items` table

2. **`app/controllers/api/v1/lti/ags/scores_controller.rb`**
   - **PRIMARY GRADE RESTORATION ENDPOINT**
   - Receives scores posted by AWS
   - Validates OAuth token and scope
   - Stores in `lti_results` table
   - **Automatically updates `self_courses` enrollment**

3. **`app/controllers/api/v1/lti/ags/results_controller.rb`**
   - Provides read access to results for AWS
   - AWS can query scores it previously posted

4. **`app/controllers/concerns/lti_token_authentication.rb`**
   - OAuth bearer token validation
   - Scope verification
   - Tool identification

---

#### NRPS Provider Flow (Provide Roster to AWS)

1. **`app/controllers/api/v1/lti/nrps/memberships_controller.rb`**
   - Provides course membership roster to AWS
   - Fetches from `event_learners` table
   - Includes basic progress data (obtained_marks, evaluation)
   - Role filtering support

---

#### AGS/NRPS Consumer Flow (Optional - TMS pulls from AWS)

1. **`app/controllers/api/v1/lti/grades_controller.rb`**
   - Manual grade fetch for admins/testing
   - Not used in normal operation

2. **`app/services/lti/grades_fetcher.rb`**
   - AGS client for pulling FROM AWS
   - Used only for manual/testing operations

3. **`app/services/lti/progress_fetcher.rb`**
   - NRPS client for pulling FROM AWS
   - Used only for manual/testing operations

---

### Configuration Files

#### `config/initializers/lti.rb`
Platform-level LTI configuration:
```ruby
module LtiConfig
  PLATFORM_ISS = ENV.fetch('LTI_PLATFORM_ISS', 'https://tms.example.com')
  KEY_PATH = Rails.root.join('config', 'lti_private_key.pem')
end
```

#### `config/routes.rb`
Route definitions for LTI endpoints:
```ruby
namespace :lti do
  get   "launch/:deployment_id", to: "launches#start"
  match "authorize", to: "launches#authorize", via: [:get, :post]
  get   "jwks", to: "jwks#show"
  post  "token", to: "token#create"
end

namespace :api do
  namespace :v1 do
    namespace :lti do
      resources :tools, only: [] do
        resources :grades, only: [:index]
        resources :progress, only: [:index]
      end
    end
  end
end
```

---

### Error Handling

All services implement comprehensive error handling:

#### Network Errors
- HTTP timeouts (15 seconds)
- Connection failures
- Invalid responses

#### Authentication Errors
- Invalid tokens
- Expired sessions
- Missing credentials

#### Validation Errors
- Missing required parameters
- Invalid context IDs
- Non-existent resources

#### Logging
Extensive logging at all stages:
- Request/response logging
- Parameter logging
- Error logging with backtraces
- Performance metrics

Log location: `app/controllers/lti/launches_controller.rb:14-24`, `app/services/lti/grades_fetcher.rb:21-42`

---

## Appendix: LTI 1.3 Standard Scopes

### AGS (Assignment and Grade Services)

| Scope | Purpose | Access Level |
|-------|---------|--------------|
| `https://purl.imsglobal.org/spec/lti-ags/scope/lineitem` | Manage line items | Read/Write |
| `https://purl.imsglobal.org/spec/lti-ags/scope/lineitem.readonly` | View line items | Read-only |
| `https://purl.imsglobal.org/spec/lti-ags/scope/result.readonly` | View results | Read-only |
| `https://purl.imsglobal.org/spec/lti-ags/scope/score` | Post scores | Write |

### NRPS (Names and Role Provisioning Services)

| Scope | Purpose | Access Level |
|-------|---------|--------------|
| `https://purl.imsglobal.org/spec/lti-nrps/scope/contextmembership.readonly` | View membership | Read-only |

---

## Glossary

- **AGS**: Assignment and Grade Services - LTI service for grade exchange
- **Context**: A course or learning environment
- **Deployment**: An instance of a tool installation
- **ID Token**: JWT containing user and context claims
- **Line Item**: An assignment or gradable activity
- **LTI**: Learning Tools Interoperability - IMS Global standard
- **NRPS**: Names and Role Provisioning Services - LTI service for roster access
- **OIDC**: OpenID Connect - Authentication protocol
- **Platform**: The LMS/TMS (this system)
- **Resource Link**: A specific piece of content within a tool
- **Result**: A grade or score for a user on a line item
- **Tool**: The external application (AWS)

---

## Contact & Support

For questions or issues regarding this integration:

**Technical Implementation**
- Location: `app/controllers/lti/` and `app/services/lti/`
- Database Schema: `db/schema.rb` (search for `lti_` tables)
- Tests: `test/lti_*_test.rb`

**Documentation References**
- LTI 1.3 Core: https://www.imsglobal.org/spec/lti/v1p3/
- LTI AGS: https://www.imsglobal.org/spec/lti-ags/v2p0/
- LTI NRPS: https://www.imsglobal.org/spec/lti-nrps/v2p0/

---

---

## Summary for AWS SkillBuild

### Key Points

1. **Grade Restoration is PUSH-based**
   - AWS **pushes** grades TO TMS automatically
   - TMS does NOT fetch grades FROM AWS
   - Endpoint: `POST /api/v1/lti/ags/lineitems/{lineitem_id}/scores`

2. **Workflow**
   - Learner launches course from TMS → AWS
   - Learner completes activity in AWS
   - AWS automatically posts score to TMS
   - TMS updates learner's enrollment record (`self_courses`)

3. **OAuth Authentication**
   - AWS requests access token from TMS: `POST /lti/token`
   - AWS includes token in score POST: `Authorization: Bearer {token}`
   - Required scope: `https://purl.imsglobal.org/spec/lti-ags/scope/score`

4. **Score Format**
   ```json
   {
     "userId": "user-{tms_user_id}",
     "scoreGiven": 85,
     "scoreMaximum": 100,
     "activityProgress": "Completed",
     "gradingProgress": "FullyGraded"
   }
   ```

5. **Automatic Updates**
   When TMS receives a score, it automatically:
   - Stores in `lti_results` table
   - Updates `self_courses.obtained_marks` (percentage)
   - Updates `self_courses.progress` (100% if completed)
   - Updates `self_courses.evaluation` (passed/failed)
   - Updates `self_courses.status` (completed if applicable)

6. **Roster Access (Optional)**
   - If AWS needs roster data, it can fetch from TMS
   - Endpoint: `GET /api/v1/lti/nrps/contexts/{context_id}/memberships`
   - Provides enrolled learner list with basic progress

---

**End of Document**
