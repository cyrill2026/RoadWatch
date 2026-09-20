# RoadWatch Initial API Contract

Design only. Login/session domains are proposed. See the completed manual appendices for normative field, session and retry details.

## EP-01 / Login (planned addition)

**HTTP Method + Route:** POST /sessions

**Purpose / Related Use Case:** Proposed FR-15 / UC-08; proposed RW-07 Login. Establish identity before viewing the dashboard.

**Authorized Roles:** Unauthenticated account holder. No public self-registration in this scope.

**Required Headers:** Content-Type: application/json; Accept: application/json. HTTPS; trusted Origin for browser request.

**Path / Query Parameters:** No path or query parameters.

**Request JSON Body:** {"username":"demo_monitor","password":"<entered password>"}; both required strings; username 1-100 chars, password 1-256 chars.

**Server-Side Validation:** Reject malformed/extra fields; verify password hash and active account; rate-limit attempts; role comes from stored account, never the client.

**Success Response:** 200: {"user":{"id":1,"username":"demo_monitor","role":"monitoring_user"}}. Set secure HttpOnly session cookie; no token in JSON.

**Error Responses:** 400 malformed JSON; 401 INVALID_CREDENTIALS (generic); 403 ORIGIN_DENIED; 415; 422; 429 RATE_LIMITED; 500.

**PostgreSQL Data Domain:** Proposed users and auth_sessions. No account/session table was evidenced in the screenshot. IT 108 extension required.

## EP-02 / Retrieve pothole records

**HTTP Method + Route:** GET /detections

**Purpose / Related Use Case:** FR-11, FR-14; UC-06; RW-03 Detection Records. Also supplies map, overview, analytics and activity.

**Authorized Roles:** monitoring_user or administrator; authorized shared pothole dataset.

**Required Headers:** Accept: application/json; Cookie: roadwatch_session=<opaque value>. HTTPS. No request Content-Type needed.

**Path / Query Parameters:** No path parameters. Optional limit: integer 1-100, default 50; offset: integer >=0, default 0.

**Request JSON Body:** No request body. Example: GET /detections?limit=50&offset=0

**Server-Side Validation:** Verify session and role; validate query; fixed Pothole filter; order detected_at DESC, id DESC.

**Success Response:** 200: {"items":[],"limit":50,"offset":0,"has_more":false}. Record object and populated example: Appendix C.

**Error Responses:** 401 AUTH_REQUIRED; 403 FORBIDDEN; 422 VALIDATION_ERROR; 500 INTERNAL_ERROR. Envelope: Appendix C.

**PostgreSQL Data Domain:** public.detections (existing); users and auth_sessions (planned access checks).

## EP-03 / Submit or retry one pothole detection

**HTTP Method + Route:** POST /detections

**Purpose / Related Use Case:** FR-09/10/12/13; UC-05/07. Mobile synchronization; not a new web entry form.

**Authorized Roles:** mobile_driver with valid session. Re-login is required before retry if session expires.

**Required Headers:** Content-Type: application/json; Accept: application/json; session cookie. Browser writes also require trusted Origin.

**Path / Query Parameters:** No path or query parameters. Stable client_detection_id identifies the queued record.

**Request JSON Body:** Required: client_detection_id string(1-100), damage_type string, confidence number, detected_at ISO-8601 with zone. Optional GPS pair: number/null. See Appendix C.

**Server-Side Validation:** Pothole only; finite confidence 0-1; valid GPS pair/ranges; timezone required; reject unknown fields. Validate auth first.

**Success Response:** 201 for first save; 200 for exact retry. {"item":<record>,"duplicate":false/true}. id is server-generated integer.

**Error Responses:** 400 malformed JSON; 401; 403; 409 ID_CONFLICT; 415 wrong media type; 422 invalid fields; 500. Safe envelope: Appendix C.

**PostgreSQL Data Domain:** public.detections. Atomic insert/check by client_detection_id. Existing uniqueness constraint must be verified before implementation.

## EP-04 / Logout (planned addition)

**HTTP Method + Route:** DELETE /sessions/current

**Purpose / Related Use Case:** Proposed FR-16 / UC-09; logout action in the authenticated dashboard.

**Authorized Roles:** Current session holder. An absent/expired session also receives 204 to permit safe repeated logout.

**Required Headers:** Cookie: roadwatch_session=<opaque value>, if present. HTTPS; trusted Origin for browser request.

**Path / Query Parameters:** No path/query parameters.

**Request JSON Body:** No request body.

**Server-Side Validation:** Validate browser Origin; revoke matching session if present. Never accept a user ID from the client for logout.

**Success Response:** 204 No Content. Expire the session cookie; Flutter clears private state and returns to Login.

**Error Responses:** 403 ORIGIN_DENIED; 500 INTERNAL_ERROR. On failure, clear local private UI but do not claim server revocation succeeded.

**PostgreSQL Data Domain:** Proposed auth_sessions; users association only. Does not delete detections or user accounts.

