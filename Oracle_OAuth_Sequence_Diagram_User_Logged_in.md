# OAuth 2.0 / OIDC Integration - Oracle MyLearn
 ## TMS-Initiated Flow (User Already Logged In)

 **Document Version:** 2.0
 **Date:** January 22, 2026
 **Prepared for:** Oracle MyLearn Team
 **Integration Type:** OAuth 2.0 + OpenID Connect (Authentication/SSO)
 **Scenario:** User logged into TMS initiates Oracle course launch

 ---

 ## Executive Summary

 This document describes the OAuth 2.0 / OpenID Connect (OIDC) authentication flow when a **user who is already logged into TMS** clicks to launch an Oracle MyLearn
 course from within the TMS platform.

 **TMS Role**: Identity Provider (IdP)
 **Oracle Role**: Service Provider (SP) / Relying Party
 **Protocol**: OAuth 2.0 (RFC 6749) + OpenID Connect Core 1.0
 **User State**: Already authenticated in TMS

 ---

 ## Integration Overview

 ### Purpose
 Enable seamless Single Sign-On (SSO) from TMS to Oracle MyLearn for users who are already authenticated.

 ### Flow Type
 **Authorization Code Flow** with **Auto-Approval** (no consent screen)

 ### Key Features
 - ✅ No re-authentication required (user already logged in)
 - ✅ No consent screen (auto-approved)
 - ✅ Seamless redirect flow (~2-3 seconds)
 - ✅ Secure token-based authentication
 - ✅ Session persistence across platforms

 ---

 ## User Journey - TMS-Initiated Sequence Diagram

 ```
 ┌─────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
 │  User   │      │     TMS      │      │Oracle MyLearn│      │  TMS User    │
 │(Already │      │  (Identity   │      │   (Service   │      │  Database    │
 │Logged In│      │   Provider)  │      │   Provider)  │      │              │
 └────┬────┘      └──────┬───────┘      └──────┬───────┘      └──────┬───────┘
      │                  │                     │                     │
      │ 1. User browses TMS platform           │                     │
      │    (already has active TMS session)    │                     │
      │                  │                     │                     │
      │ 2. Click "Launch Oracle Course" button │                     │
      ├─────────────────>│                     │                     │
      │                  │                     │                     │
      │                  │ 3. Check user session                     │
      │                  │    ✅ User authenticated                  │
      │                  │    (session exists)                       │
      │                  │                     │                     │
      │ 4. Redirect to Oracle course URL       │                     │
      │<─────────────────┤                     │                     │
      ├────────────────────────────────────────>│                     │
      │                                        │                     │
      │                                        │ 5. Oracle checks session
      │                                        │    (no Oracle session found)
      │                                        │                     │
      │                                        │ 6. Redirect to TMS for auth
      │                                        │    GET /oauth/authorize
      │                                        │    ?client_id=<oracle_client_id>
      │                                        │    &redirect_uri=<oracle_callback>
      │                                        │    &response_type=code
      │                                        │    &scope=openid+email+profile
      │                                        │    &state=<random_csrf_token>
      │<────────────────────────────────────────┤                     │
      ├─────────────────>│                     │                     │
      │                  │                     │                     │
      │                  │ 7. Validate OAuth request                 │
      │                  │    - Check client_id                      │
      │                  │    - Validate redirect_uri                │
      │                  │    - Validate scopes                      │
      │                  │                     │                     │
      │                  │ 8. Check user session                     │
      │                  │    ✅ USER ALREADY AUTHENTICATED!         │
      │                  │    (Skip login form)                      │
      │                  │                     │                     │
      │                  │ 9. Check auto-approval                    │
      │                  │    ✅ Oracle = auto-approved              │
      │                  │    (Skip consent screen)                  │
      │                  │                     │                     │
      │                  │ 10. Generate authorization code           │
      │                  │     (single-use, 10-min expiry)           │
      │                  │                     │                     │
      │ 11. Redirect back to Oracle            │                     │
      │     with authorization code            │                     │
      │     <oracle_callback>?code=<auth_code> │                     │
      │                        &state=<same_state>                   │
      │<─────────────────┤                     │                     │
      ├────────────────────────────────────────>│                     │
      │                                        │                     │
      │                                        │ 12. Validate state parameter
      │                                        │     (CSRF protection)
      │                                        │                     │
      │                                        │ 13. Exchange code for tokens
      │                                        │     POST /oauth/token
      │                                        │     grant_type=authorization_code
      │                                        │     code=<auth_code>
      │                                        │     client_id=<oracle_client_id>
      │                                        │     client_secret=<oracle_secret>
      │                                        │     redirect_uri=<oracle_callback>
      │                                        ├────────────────────>│
      │                                        │                     │
      │                                        │ 14. Validate code   │
      │                                        │     - Not expired?  │
      │                                        │     - Not used?     │
      │                                        │     - Client valid? │
      │                                        │                     │
      │                                        │ 15. Fetch user data │
      │                                        │<────────────────────┤
      │                                        │                     │
      │                                        │ 16. Generate tokens │
      │                                        │     - Access Token  │
      │                                        │     - ID Token (JWT)│
      │                                        │     - Refresh Token │
      │                                        │                     │
      │                                        │ 17. Return tokens   │
      │                                        │     {               │
      │                                        │       access_token, │
      │                                        │       id_token,     │
      │                                        │       refresh_token,│
      │                                        │       token_type: "Bearer",
      │                                        │       expires_in: 7200
      │                                        │     }               │
      │                                        │<────────────────────┤
      │                                        │                     │
      │                                        │ 18. Validate ID Token signature
      │                                        │     - Fetch JWKS    │
      │                                        │     - Verify RS256  │
      │                                        │     - Check expiry  │
      │                                        │                     │
      │                                        │ 19. Extract user claims
      │                                        │     {               │
      │                                        │       sub: "12345", │
      │                                        │       email: "user@example.com",
      │                                        │       name: "John Doe",
      │                                        │       email_verified: true
      │                                        │     }               │
      │                                        │                     │
      │                                        │ 20. Create/update Oracle user
      │                                        │     - Link to TMS user ID (sub)
      │                                        │     - Update profile data
      │                                        │                     │
      │                                        │ 21. Create Oracle session
      │                                        │     - Store tokens  │
      │                                        │     - Set session cookie
      │                                        │                     │
      │ 22. Grant access to Oracle course      │                     │
      │     (user lands on course page)        │                     │
      │<────────────────────────────────────────┤                     │
      │                                        │                     │
      │ 23. User accesses Oracle course        │                     │
      │     (fully authenticated, seamless)    │                     │
      │                                        │                     │
 ```

 **Total Time:** ~2-3 seconds (all redirects happen automatically in background)

 ---

 ## Detailed Step-by-Step Flow

 ### Phase 1: User Initiates from TMS (Steps 1-4)

 **Step 1: User Already Logged Into TMS**
 - User has active TMS session (JWT token or session cookie)
 - Browsing courses in TMS platform

 **Step 2: User Clicks Launch**
 User clicks button/link: "Launch Oracle Course"

 **Frontend Example:**
 ```typescript
 <button onClick={() => window.location.href = 'https://oracle-mylearn.com/course/12345'}>
   Launch Oracle Course
 </button>
 ```

 **Step 3: TMS Validates Session**
 ```ruby
 # TMS backend checks
 current_user.present?  # => true (user authenticated)
 ```

 **Step 4: Redirect to Oracle**
 TMS redirects browser to Oracle course URL:
 ```
 https://oracle-mylearn.com/course/12345
 ```

 ---

 ### Phase 2: Oracle Initiates OAuth Flow (Steps 5-6)

 **Step 5: Oracle Checks Session**
 - Oracle checks for existing Oracle session
 - No session found (user not logged into Oracle)

 **Step 6: Oracle Redirects to TMS**
 Oracle constructs OAuth authorization URL:

 ```
 https://your-tms-domain.com/oauth/authorize?
   client_id=ORACLE_CLIENT_ID
   &redirect_uri=https://oracle-mylearn.com/auth/callback
   &response_type=code
   &scope=openid+email+profile
   &state=abc123random
 ```

 ---

 ### Phase 3: TMS Auto-Authorization (Steps 7-10) ⭐ KEY DIFFERENCE

 **Step 7: TMS Validates OAuth Request**
 ```ruby
 # app/controllers/oauth/authorizations_controller.rb
 def new
   # Validate client_id
   @client = Doorkeeper::Application.find_by(uid: params[:client_id])
   return render_error unless @client

   # Validate redirect_uri
   return render_error unless @client.redirect_uri.include?(params[:redirect_uri])
 end
 ```

 **Step 8: Check User Session** ⭐
 ```ruby
 # Critical: User is already logged in!
 if current_user
   # ✅ Skip login form entirely
   # Proceed directly to authorization
 else
   # ❌ Would show login form (not this scenario)
   redirect_to new_user_session_path
 end
 ```

 **Step 9: Check Auto-Approval** ⭐
 ```ruby
 # config/initializers/doorkeeper.rb
 Doorkeeper.configure do
   skip_authorization do |resource_owner, client|
     # ✅ Oracle is auto-approved (no consent screen)
     client.name == 'Oracle MyLearn'
   end
 end
 ```

 **Result:** Both login and consent screens are skipped!

 **Step 10: Generate Authorization Code**
 ```ruby
 # Create single-use code
 access_grant = Doorkeeper::AccessGrant.create!(
   application: @client,
   resource_owner_id: current_user.id,
   redirect_uri: params[:redirect_uri],
   expires_in: 600,  # 10 minutes
   scopes: params[:scope]
 )

 authorization_code = access_grant.token
 ```

 ---

 ### Phase 4: Callback to Oracle (Step 11)

 **Step 11: TMS Redirects to Oracle**
 Immediate redirect (no user interaction needed):

 ```
 https://oracle-mylearn.com/auth/callback?
   code=a1b2c3d4e5f6g7h8i9j0
   &state=abc123random
 ```

 ---

 ### Phase 5: Token Exchange (Steps 12-17)

 **Steps 12-17:** Same as original flow (back-channel communication)

 Oracle exchanges code for tokens via back-channel POST to TMS.

 ---

 ### Phase 6: Oracle Session Creation (Steps 18-23)

 **Steps 18-21:** Oracle validates tokens and creates session

 **Step 22-23: User Lands on Oracle Course**
 - User seamlessly lands on Oracle course page
 - No interruption to user experience
 - Entire flow takes 2-3 seconds

 ---

 ## User Experience Comparison

 ### What User Sees (TMS-Initiated)

 ```
 1. User clicks "Launch Oracle Course" in TMS
    ↓
 2. [Brief loading spinner - 2-3 seconds]
    ↓
 3. Oracle course page opens (user is logged in)
 ```

 **User does NOT see:**
 - ❌ Login form
 - ❌ Password prompt
 - ❌ "Authorize Oracle?" consent screen
 - ❌ Any manual steps

 **User DOES see:**
 - ✅ Brief loading/redirect indicators
 - ✅ Oracle course page (destination)

 ---

 ### What User Sees (Oracle-Initiated - Original Flow)

 ```
 1. User goes directly to oracle-mylearn.com
    ↓
 2. [Redirects to TMS]
    ↓
 3. IF TMS session exists: [Auto-authorize]
    IF TMS session expired: [Login form]
    ↓
 4. Oracle course page opens
 ```

 ---

 ## Frontend Implementation

 ### Option 1: Direct Link (Simplest)

 ```typescript
 // TMS Frontend - Course Card Component
 interface OracleCourseProps {
   courseId: string;
   title: string;
   oracleUrl: string;
 }

 export const OracleCourseCard: React.FC<OracleCourseProps> = ({
   courseId,
   title,
   oracleUrl
 }) => {
   return (
     <div className="course-card">
       <h3>{title}</h3>
       <a
         href={oracleUrl}
         target="_blank"
         rel="noopener noreferrer"
         className="btn btn-primary"
       >
         Launch Oracle Course
       </a>
     </div>
   );
 };
 ```

 **Pros:** Simple, no JavaScript needed
 **Cons:** User experiences redirect delay

 ---

 ### Option 2: With Loading State (Better UX)

 ```typescript
 import React, { useState } from 'react';

 interface OracleCourseProps {
   courseId: string;
   title: string;
   oracleUrl: string;
 }

 export const OracleCourseCard: React.FC<OracleCourseProps> = ({
   courseId,
   title,
   oracleUrl
 }) => {
   const [launching, setLaunching] = useState(false);

   const handleLaunch = () => {
     setLaunching(true);

     // Open Oracle in new tab
     const oracleWindow = window.open(oracleUrl, '_blank');

     // Reset after delay
     setTimeout(() => {
       setLaunching(false);
       // Focus on Oracle window
       oracleWindow?.focus();
     }, 3000);
   };

   return (
     <div className="course-card">
       <h3>{title}</h3>

       <button
         onClick={handleLaunch}
         disabled={launching}
         className="btn btn-primary"
       >
         {launching ? (
           <>
             <Spinner className="mr-2" />
             Launching Oracle Course...
           </>
         ) : (
           'Launch Oracle Course'
         )}
       </button>

       {launching && (
         <p className="text-sm text-gray-600 mt-2">
           Securely authenticating you to Oracle MyLearn...
         </p>
       )}
     </div>
   );
 };
 ```

 **Pros:** Better UX with loading feedback
 **Cons:** Slightly more complex

 ---

 ### Option 3: Pre-Authorization Check (Advanced)

 ```typescript
 import React, { useState } from 'react';
 import axios from 'axios';

 export const OracleCourseCard: React.FC<OracleCourseProps> = ({
   courseId,
   title,
   oracleUrl
 }) => {
   const [launching, setLaunching] = useState(false);
   const [error, setError] = useState<string | null>(null);

   const handleLaunch = async () => {
     try {
       setLaunching(true);
       setError(null);

       // Check TMS session validity
       const sessionCheck = await axios.get('/api/v1/auth/check_session');

       if (!sessionCheck.data.authenticated) {
         // Session expired - show message
         setError('Session expired. Please refresh the page.');
         setLaunching(false);
         return;
       }

       // Session valid - proceed to Oracle
       const oracleWindow = window.open(oracleUrl, '_blank');

       setTimeout(() => {
         setLaunching(false);
         oracleWindow?.focus();
       }, 3000);

     } catch (err) {
       setError('Failed to launch Oracle course. Please try again.');
       setLaunching(false);
     }
   };

   return (
     <div className="course-card">
       <h3>{title}</h3>

       <button
         onClick={handleLaunch}
         disabled={launching}
         className="btn btn-primary"
       >
         {launching ? 'Launching...' : 'Launch Oracle Course'}
       </button>

       {error && (
         <p className="text-red-600 text-sm mt-2">{error}</p>
       )}
     </div>
   );
 };
 ```

 **Pros:** Validates session before launch, better error handling
 **Cons:** Extra API call, more complex

 ---

 ## Backend Configuration

 ### 1. Enable Auto-Approval for Oracle

 ```ruby
 # config/initializers/doorkeeper.rb
 Doorkeeper.configure do
   # Skip authorization prompt for trusted clients
   skip_authorization do |resource_owner, client|
     # Oracle is trusted - no consent screen needed
     client.name == 'Oracle MyLearn'
   end

   # Optional: Skip re-authentication if session exists
   resource_owner_from_credentials do |routes|
     # Check for existing session
     user = User.find_by(id: session[:user_id]) if session[:user_id]
     user if user&.active?
   end
 end
 ```

 ---

 ### 2. Create Oracle OAuth Client

 ```ruby
 # Rails console or seed file
 oracle_client = Doorkeeper::Application.create!(
   name: 'Oracle MyLearn',
   redirect_uri: 'https://oracle-mylearn.com/auth/callback',
   scopes: 'openid email profile',
   confidential: true  # Client secret required
 )

 puts "Oracle Client ID: #{oracle_client.uid}"
 puts "Oracle Client Secret: #{oracle_client.secret}"
 ```

 ---

 ### 3. Configure Session Persistence

 ```ruby
 # config/initializers/session_store.rb
 Rails.application.config.session_store :cookie_store,
   key: '_tms_session',
   expire_after: 24.hours,  # Keep session alive
   secure: Rails.env.production?,
   same_site: :lax  # Allow cross-site redirects
 ```

 ---

 ## Security Considerations

 ### Session Security

 **TMS Session:**
 - ✅ Secure cookies (HTTPS only in production)
 - ✅ Same-site: Lax (allows OAuth redirects)
 - ✅ HTTP-only cookies (JavaScript cannot access)
 - ✅ Session expiration (24 hours default)

 **State Parameter:**
 - ✅ CSRF protection (Oracle generates, TMS echoes)
 - ✅ Validated on callback
 - ✅ Single-use

 **Authorization Code:**
 - ✅ Single-use only
 - ✅ 10-minute expiration
 - ✅ Bound to client and user
 - ✅ Deleted after token exchange

 ---

 ### Auto-Approval Risks

 **Low Risk for Oracle:**
 - ✅ Trusted partner (Oracle MyLearn)
 - ✅ Limited scopes (openid, email, profile)
 - ✅ No write permissions
 - ✅ Read-only user data

 **Best Practice:**
 ```ruby
 # Only auto-approve specific trusted clients
 skip_authorization do |resource_owner, client|
   # Whitelist approach
   trusted_clients = ['Oracle MyLearn', 'Trusted Partner X']
   trusted_clients.include?(client.name)
 end
 ```

 ---

 ## Timing Breakdown

 ### Redirect Flow Duration

 | Step | Action | Time |
 |------|--------|------|
 | 1 | User clicks button | 0ms |
 | 2 | Redirect TMS → Oracle | 200-300ms |
 | 3 | Oracle checks session | 100-200ms |
 | 4 | Redirect Oracle → TMS | 200-300ms |
 | 5 | TMS validates OAuth request | 100-200ms |
 | 6 | TMS checks user session ✅ | 50-100ms |
 | 7 | TMS checks auto-approval ✅ | 10-50ms |
 | 8 | TMS generates auth code | 50-100ms |
 | 9 | Redirect TMS → Oracle | 200-300ms |
 | 10 | Oracle exchanges code | 300-500ms |
 | 11 | Oracle validates & creates session | 200-300ms |
 | 12 | Oracle renders course page | 500-1000ms |
 | **Total** | | **~2-3 seconds** |

 ---

 ## Error Scenarios

 ### Scenario 1: TMS Session Expired

 ```
 User clicks "Launch Oracle"
     ↓
 TMS session check fails
     ↓
 Redirect to TMS login page
     ↓
 User enters credentials
     ↓
 After login → continue OAuth flow
     ↓
 Oracle course opens
 ```

 **Frontend Handling:**
 ```typescript
 // Detect session expiration
 axios.interceptors.response.use(
   response => response,
   error => {
     if (error.response?.status === 401) {
       // Session expired - redirect to login
       window.location.href = '/login?return_to=' + encodeURIComponent(window.location.href);
     }
     return Promise.reject(error);
   }
 );
 ```

 ---

 ### Scenario 2: Oracle Client Not Configured

 ```
 Oracle redirects to TMS
     ↓
 TMS: "Invalid client_id"
     ↓
 Error: "Client authentication failed"
 ```

 **Fix:**
 ```ruby
 # Verify Oracle client exists
 Doorkeeper::Application.find_by(name: 'Oracle MyLearn')
 # => Should not be nil
 ```

 ---

 ### Scenario 3: Invalid Redirect URI

 ```
 Oracle sends redirect_uri not in whitelist
     ↓
 TMS rejects OAuth request
     ↓
 Error: "Redirect URI mismatch"
 ```

 **Fix:**
 ```ruby
 # Update Oracle client redirect URIs
 oracle_client = Doorkeeper::Application.find_by(name: 'Oracle MyLearn')
 oracle_client.update!(
   redirect_uri: "https://oracle-mylearn.com/auth/callback\nhttps://oracle-staging.com/auth/callback"
 )
 ```

 ---

 ## Testing Checklist

 ### Pre-Integration Testing

 - [ ] TMS session persists across redirects
 - [ ] Oracle client configured with correct redirect URIs
 - [ ] Auto-approval enabled for Oracle
 - [ ] OIDC discovery endpoint accessible
 - [ ] JWKS endpoint accessible

 ### User Flow Testing

 **Test 1: User Logged Into TMS**
 - [ ] User clicks "Launch Oracle Course"
 - [ ] No login form shown
 - [ ] No consent screen shown
 - [ ] User lands on Oracle course (~2-3 sec)

 **Test 2: User NOT Logged Into TMS**
 - [ ] User clicks "Launch Oracle Course"
 - [ ] Redirected to TMS login
 - [ ] After login → continues to Oracle
 - [ ] No consent screen shown

 **Test 3: TMS Session Expired**
 - [ ] User clicks launch
 - [ ] Redirected to TMS login
 - [ ] After login → continues to Oracle

 ---

 ## Comparison: TMS-Initiated vs Oracle-Initiated

 | Aspect | **TMS-Initiated** (This Doc) | **Oracle-Initiated** (Original) |
 |--------|------------------------------|--------------------------------|
 | **Entry Point** | User starts in TMS | User starts in Oracle |
 | **User State** | Already logged into TMS | May or may not be logged in |
 | **Login Form** | Never shown (if session valid) | Shown if TMS session expired |
 | **Redirect Count** | 3-4 redirects | 3-4 redirects |
 | **Total Time** | ~2-3 seconds | ~2-3 seconds (or more if login needed) |
 | **User Experience** | Seamless (if logged in) | May require login |
 | **Use Case** | TMS has Oracle course listings | Oracle is standalone portal |

 ---

 ## Recommended Implementation

 ### 1. TMS Course Catalog with Oracle Courses

 ```typescript
 // TMS Frontend - Course Listing Page
 import { OracleCourseCard } from './components/OracleCourseCard';

 function CourseCatalog() {
   const courses = [
     {
       id: 1,
       type: 'external',
       provider: 'Oracle MyLearn',
       title: 'Oracle Database Administration',
       oracleUrl: 'https://oracle-mylearn.com/course/db-admin-101'
     },
     {
       id: 2,
       type: 'external',
       provider: 'Oracle MyLearn',
       title: 'Java SE Programming',
       oracleUrl: 'https://oracle-mylearn.com/course/java-se-101'
     }
   ];

   return (
     <div className="course-catalog">
       <h1>Available Courses</h1>
       <div className="course-grid">
         {courses.map(course => (
           <OracleCourseCard key={course.id} course={course} />
         ))}
       </div>
     </div>
   );
 }
 ```

 ---

 ### 2. Database Schema for Oracle Courses

 ```ruby
 # Migration
 class AddOracleCoursesToCourses < ActiveRecord::Migration[7.0]
   def change
     add_column :courses, :oracle_course_url, :string
     add_column :courses, :oracle_course_id, :string
     add_index :courses, :oracle_course_id
   end
 end

 # Model
 class Course < ApplicationRecord
   enum course_type: { internal: 0, external: 1, oracle: 2 }

   def oracle_course?
     course_type == 'oracle' && oracle_course_url.present?
   end

   def launch_url
     case course_type
     when 'oracle'
       oracle_course_url
     when 'external'
       # LTI launch URL for IBM/AWS
       "/lti/launch/#{lti_deployment_id}?context_id=#{id}"
     else
       # Internal course
       "/courses/#{id}"
     end
   end
 end
 ```

 ---

 ### 3. API Endpoint for Course Launch

 ```ruby
 # app/controllers/api/v1/courses_controller.rb
 class Api::V1::CoursesController < ApplicationController
   before_action :authenticate_user!

   def launch
     course = Course.find(params[:id])

     case course.course_type
     when 'oracle'
       # Return Oracle URL - frontend handles redirect
       render json: {
         type: 'redirect',
         url: course.oracle_course_url,
         provider: 'Oracle MyLearn',
         message: 'Redirecting to Oracle MyLearn...'
       }
     when 'external'
       # LTI launch URL for IBM/AWS
       render json: {
         type: 'lti_launch',
         url: course.lti_launch_url(current_user),
         provider: course.provider
       }
     else
       # Internal course
       render json: {
         type: 'internal',
         url: "/courses/#{course.id}"
       }
     end
   end
 end
 ```

 ---

 ## Summary

 ### Key Differences from Original Flow

 | Original (Oracle-Initiated) | TMS-Initiated (This Document) |
 |----------------------------|-------------------------------|
 | User browses to Oracle first | User browses to TMS first |
 | Oracle initiates OAuth | TMS redirects → Oracle initiates OAuth |
 | User may see login form | User never sees login (if logged in) |
 | Use case: Oracle standalone | Use case: TMS course catalog |

 ### What Makes This Seamless

 1. ✅ **User already authenticated** - No login form
 2. ✅ **Auto-approval enabled** - No consent screen
 3. ✅ **Session persistence** - TMS session valid across redirects
 4. ✅ **Loading states** - Frontend shows progress
 5. ✅ **Fast redirects** - 2-3 seconds total

 ### Best Practice

 **Show this to users:**
 ```
 "Launching Oracle Course...
 Securely connecting you to Oracle MyLearn."
 ```

 This sets proper expectations for the brief redirect flow.

 ---

 **Document Version:** 2.0
 **Last Updated:** January 22, 2026
 **Protocol:** OAuth 2.0 (RFC 6749) + OpenID Connect Core 1.0
 **Scenario:** TMS-initiated launch with existing user session
