# Implementation Plan: RouteGuard

## Overview

RouteGuard is implemented as an event-driven serverless application on AWS using TypeScript Lambda functions, DynamoDB for persistence, SNS for push notifications, Bedrock (Claude) for plain-language alert generation, and OneMap APIs for BFA routing and geocoding. Implementation proceeds bottom-up: shared utilities and data layer first, then individual service Lambdas, then integration and wiring, and finally the companion web dashboard.

## Tasks

- [ ] 1. Set up project structure, shared utilities, and DynamoDB table definitions
  - [ ] 1.1 Initialize project with TypeScript, AWS CDK (or SAM), and shared config
    - Create project directory structure: `infra/`, `src/lambdas/`, `src/shared/`, `tests/unit/`, `tests/property/`, `tests/integration/`
    - Initialize `package.json` with TypeScript, AWS SDK v3, fast-check, vitest, and CDK/SAM dependencies
    - Configure `tsconfig.json` for Lambda-compatible output (ES2020, CommonJS)
    - Create shared constants file with Singapore coordinate bounds, disruption type enum, geohash precision, and route/caregiver limits
    - _Requirements: 1.1, 4.4, 7.6_

  - [ ] 1.2 Define DynamoDB table schemas in IaC (CDK or SAM template)
    - Define Disruptions table with PK `disruptionId`, GSI `geohash-status-index` (PK: `geohash`, SK: `status`)
    - Define User Profiles table with PK `userId` (stores saved routes as embedded list with full geometry)
    - Define Risk Scores table with PK `locationNodeId`
    - Define Alert Deduplication table with PK `userId#routeId#disruptionId` and TTL attribute
    - Enable DynamoDB Stream (NEW_IMAGE) on Disruptions table
    - _Requirements: 1.3, 1.6, 4.3, 5.1_

  - [ ] 1.3 Implement shared geospatial utility functions
    - Implement Haversine distance calculation between two coordinate pairs
    - _Requirements: 1.6, 2.1_

  - [ ]* 1.4 Write property test for geospatial utilities
    - **Property 3: Route-disruption proximity matching** (Haversine distance portion)
    - Generate random coordinate pairs and verify Haversine distance is non-negative, symmetric, and zero when points are identical
    - **Validates: Requirements 2.1**

- [ ] 2. Implement Disruption Ingestion Service
  - [ ] 2.1 Implement disruption payload validation logic
    - Create validation function that checks latitude [1.15, 1.47], longitude [103.60, 104.05], disruptionType enum membership, and ISO 8601 timestamp
    - Return field-level error details for any validation failures
    - _Requirements: 1.1, 1.2_

  - [ ]* 2.2 Write property test for disruption payload validation
    - **Property 1: Disruption payload validation**
    - Generate random payloads (both valid and invalid), verify validation accepts if and only if all fields are valid, and rejects with correct field identification otherwise
    - **Validates: Requirements 1.1, 1.2**

  - [ ] 2.3 Implement disruption ingestion Lambda handler
    - Handle `POST /disruptions`: validate payload, compute geohash, store in DynamoDB with status "active" and generated UUID
    - Handle `PUT /disruptions/{disruptionId}/resolve`: lookup disruption, update status to "resolved" with resolution timestamp; return 404 if not found
    - Handle `GET /disruptions?status=active`: query GSI for active disruptions (for dashboard)
    - Return appropriate HTTP status codes (201, 400, 404, 500)
    - Log DynamoDB write failures for operational monitoring
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.7, 1.9_

  - [ ]* 2.4 Write property test for valid disruption storage invariant
    - **Property 2: Valid disruption storage invariant**
    - Generate valid payloads, verify stored record has status "active", contains geohash, and preserves all input fields
    - **Validates: Requirements 1.3, 1.4, 1.6**

  - [ ] 2.5 Create seed data script with sample disruption records
    - Create script that inserts at least 10 sample disruptions modeled on LTA and SMRT lift-status report formats
    - Include a mix of disruption types and locations spread across Singapore
    - _Requirements: 1.8_

- [ ] 3. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement User Authentication and Profile Management
  - [ ] 4.1 Configure AWS Cognito user pool in IaC
    - Define Cognito User Pool with email/phone sign-up
    - Configure API Gateway authorizer using Cognito JWT tokens
    - Define protected endpoints (all except health check)
    - _Requirements: 7.1, 7.2_

  - [ ] 4.2 Implement user profile creation and management
    - On Cognito post-confirmation trigger, create User_Profile in DynamoDB linked to Cognito identity
    - Implement caregiver contact validation: E.164 phone format or standard email format (local-part@domain)
    - Enforce maximum 3 caregivers per user
    - Return field-level errors on invalid caregiver contact
    - _Requirements: 7.3, 7.4, 7.5, 7.6, 7.7_

  - [ ]* 4.3 Write property test for caregiver contact validation
    - **Property 13: Caregiver contact validation**
    - Generate random contact strings, verify validation accepts E.164 phone or valid email, and rejects with correct field identification otherwise
    - **Validates: Requirements 7.4, 7.5**

- [ ] 5. Implement User Route Management
  - [ ] 5.1 Implement geocoding and BFA route search
    - Call OneMap Geocoding API with address input
    - If multiple results: return top 5 for user selection
    - If zero results: return error prompting user to refine input
    - Call OneMap BFA Routing endpoint once coordinates are resolved
    - Handle OneMap API unavailability with user-facing error message
    - _Requirements: 4.1, 4.2, 4.7, 4.8, 4.9_

  - [ ] 5.2 Implement route save, list, and delete operations
    - `POST /routes`: Validate route data completeness (geometry, origin, destination, name, timestamp), enforce max 10 routes, store full route geometry in User Profile
    - `GET /routes`: Return user's saved routes from User Profile
    - `DELETE /routes/{routeId}`: Remove route from User Profile
    - _Requirements: 4.3, 4.4, 4.5, 4.6_

  - [ ]* 5.3 Write property test for saved route data completeness
    - **Property 9: Saved route data completeness**
    - Generate route data, verify all required fields (geometry, origin address, destination address, route name, creation timestamp) are persisted and non-null
    - **Validates: Requirements 4.3**

  - [ ]* 5.4 Write property test for route deletion ceases monitoring
    - **Property 10: Route deletion ceases monitoring**
    - Generate routes then delete them, verify route is removed from User Profile
    - **Validates: Requirements 4.6**

- [ ] 6. Implement Route Matcher
  - [ ] 6.1 Implement Route Matcher Lambda triggered by DynamoDB Stream
    - Listen for INSERT events on Disruptions table (active disruptions only)
    - Extract disruption location (latitude, longitude)
    - Scan all User Profiles and iterate every coordinate point of every saved BFA_Route
    - Compute Haversine distance from disruption to each coordinate point
    - Filter to users with at least one route coordinate within 50m of disruption
    - Invoke Alert Service for each affected user with disruption + route context
    - On failure: log error, retry once, then mark disruption as `unmatched`
    - _Requirements: 2.1, 2.2, 2.3, 2.5, 2.6_

  - [ ] 6.2 Implement new-route check against active disruptions
    - When a new BFA_Route is saved (triggered after route storage in 5.2), query all active disruptions and compute Haversine distance to each coordinate of the new route
    - Alert user if any active disruptions are within 50m of the new route
    - _Requirements: 2.4_

  - [ ]* 6.3 Write property test for route-disruption proximity matching
    - **Property 3: Route-disruption proximity matching**
    - Generate random disruption locations and route coordinate arrays, verify exactly those routes with at least one coordinate within 50m are matched
    - **Validates: Requirements 2.1, 2.2, 2.4**

  - [ ]* 6.4 Write property test for resolved disruptions excluded from matching
    - **Property 4: Resolved disruptions excluded from matching**
    - Generate resolved disruptions, verify they are never included in matching regardless of geographic proximity
    - **Validates: Requirements 2.6**

- [ ] 7. Implement Alert Service and Message Generator
  - [ ] 7.1 Implement Alert Service Lambda
    - Accept AlertRequest (userId, routeId, disruption, departureStatus, userLocation)
    - Determine departure status: "pre_departure" if user within 100m of route origin, "en_route" otherwise
    - For en-route alerts: only send if disruption is ≥200m ahead of user position along the route
    - Invoke Alert_Message_Generator (Bedrock) with disruption context
    - If Bedrock exceeds 5s, fall back to template-based message
    - Check alert deduplication table before sending; skip if already alerted for same (userId, routeId, disruptionId) unless status changed
    - Include deep link to map view centered on disruption coordinates in notification payload
    - Publish to SNS platform endpoint for the user's device
    - Send within 60 seconds of match being identified
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7_

  - [ ] 7.2 Implement Alert Message Generator (Bedrock Claude integration)
    - Construct prompt from disruption data using the defined prompt template
    - Invoke AWS Bedrock (Claude) with 5s timeout
    - Validate output: ≤450 characters, includes what/where/what-to-do
    - Handle missing input fields by indicating unavailable information in the message
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6_

  - [ ] 7.3 Implement fallback template message generator
    - Produce structured template message when Bedrock times out
    - Ensure template output is ≤450 characters and includes what happened, where, and what to do next
    - Handle cases where disruption data fields are missing
    - _Requirements: 3.5, 8.4, 8.6_

  - [ ]* 7.4 Write property test for departure status classification
    - **Property 5: Departure status classification**
    - Generate random user positions and route origins, verify "pre_departure" if and only if Haversine distance ≤ 100m
    - **Validates: Requirements 3.7**

  - [ ]* 7.5 Write property test for en-route alert distance threshold
    - **Property 6: En-route alert distance threshold**
    - Generate en-route positions and disruption locations along a route, verify alert is sent if and only if disruption is ≥200m ahead
    - **Validates: Requirements 3.2**

  - [ ]* 7.6 Write property test for alert deduplication
    - **Property 7: Alert deduplication**
    - Generate duplicate matching events for the same (userId, routeId, disruptionId), verify only one alert is sent unless disruption status changes
    - **Validates: Requirements 3.6**

  - [ ]* 7.7 Write property test for alert notification deep link
    - **Property 8: Alert notification deep link**
    - Generate random disruption coordinates, verify notification payload contains a deep link with latitude and longitude
    - **Validates: Requirements 3.4**

  - [ ]* 7.8 Write property test for alert message output constraints
    - **Property 14: Alert message output constraints**
    - Generate disruption data with random missing fields, verify output is ≤450 characters, includes required elements when available, and indicates unavailability for missing fields
    - **Validates: Requirements 8.2, 8.4, 8.6**

- [ ] 8. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement Risk Score Engine
  - [ ] 9.1 Implement Risk Score Engine Lambda
    - Trigger on DynamoDB Stream when disruption status changes to "resolved": increment disruption count for affected Location_Node, record resolution timestamp
    - Trigger on EventBridge daily schedule: recalculate all Location_Node risk scores excluding disruptions older than 90 days
    - Calculate risk score: `min(100, (count_30d × 2) + (count_31_90d × 1))`
    - Assign score of 0 when no disruptions in past 90 days
    - Expose `GET /routes/{routeId}/risk-scores` endpoint returning scores for all Location_Nodes along the route
    - Return "unavailable" indicator on calculation failure
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_

  - [ ]* 9.2 Write property test for risk score calculation correctness
    - **Property 11: Risk score calculation correctness**
    - Generate disruption histories with various resolved timestamps, verify score equals `min(100, (count_30d × 2) + (count_31_90d × 1))`, always in [0, 100], and excludes >90 day disruptions
    - **Validates: Requirements 5.1, 5.2, 5.4, 5.5**

- [ ] 10. Implement Stranded Mode Service
  - [ ] 10.1 Implement Stranded Mode Lambda
    - Handle `POST /sos` with current location and optional activeRouteId
    - Request BFA reroute from OneMap (current location → original destination); skip if no active route
    - Send caregiver notifications via SNS: each notification includes map link of user location, timestamp, and SOS message text; skip if no caregivers configured
    - Query OneMap thematic layer ("Programmes and Services for Persons with Disabilities") for Help_Points within 500m; expand to 1000m if zero found; return up to 5 nearest
    - Execute reroute, notification, and Help_Point lookup in parallel where possible
    - Return combined response within 10 seconds
    - Handle error states: no location → prompt to enable location services, reroute fails → show Help_Points + caregiver status, no caregivers → skip and inform user
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.9_

  - [ ]* 10.2 Write property test for caregiver notification on SOS
    - **Property 12: Caregiver notification on SOS**
    - Generate users with 1–3 caregivers, verify exactly N notifications sent, each containing map link, timestamp, and SOS message
    - **Validates: Requirements 6.3**

- [ ] 11. Wire API Gateway, configure endpoints, and integrate all services
  - [ ] 11.1 Define API Gateway REST API with all endpoints and Cognito authorizer
    - Configure endpoints: `POST /disruptions`, `PUT /disruptions/{disruptionId}/resolve`, `GET /disruptions?status=active`, `GET /routes/search`, `POST /routes`, `DELETE /routes/{routeId}`, `GET /routes`, `GET /routes/{routeId}/risk-scores`, `POST /sos`
    - Attach Cognito authorizer to all protected endpoints
    - Configure Lambda integrations for each endpoint
    - Configure DynamoDB Stream trigger from Disruptions table to Route Matcher Lambda
    - Configure EventBridge rule for daily Risk Score Engine invocation
    - _Requirements: 7.1, 7.2_

  - [ ] 11.2 Implement end-to-end disruption → alert pipeline integration
    - Verify DynamoDB Stream correctly triggers Route Matcher
    - Verify Route Matcher correctly invokes Alert Service
    - Verify Alert Service correctly publishes to SNS
    - Wire error handling and dead-letter queues (SQS DLQ for failed SNS, unmatched disruptions table)
    - _Requirements: 2.1, 2.2, 3.1_

- [ ] 12. Implement Companion Web Dashboard
  - [ ] 12.1 Set up React web dashboard project with Amplify Hosting configuration
    - Initialize React project with Vite build
    - Configure Leaflet or MapLibre GL JS with OneMap basemap tiles
    - Configure AWS Amplify Hosting for static site deployment (CI/CD from Git)
    - Set up Cognito SDK authentication (shared user pool or read-only demo token)
    - _Requirements: N/A (competition demo requirement from design)_

  - [ ] 12.2 Implement dashboard map features
    - Display saved BFA_Routes as polylines on the interactive map
    - Show active disruptions as map markers with type-specific icons (10s polling refresh)
    - Overlay risk score values on Location_Nodes along displayed routes, color-coded by severity (green 0–30, amber 31–60, red 61–100)
    - Connect to API Gateway endpoints: `GET /routes`, `GET /disruptions?status=active`, `GET /routes/{routeId}/risk-scores`
    - _Requirements: N/A (competition demo requirement from design)_

- [ ] 13. Final checkpoint - Ensure all tests pass and deploy
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using fast-check (TypeScript)
- Unit tests validate specific examples and edge cases
- Route matching uses brute-force Haversine distance checking against all saved route coordinates (no geohash indexing) — acceptable at MVP scale with a small user/route count
- The web dashboard (tasks 12.x) is for competition demo purposes and reads from the same API endpoints as the mobile app

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "1.3"] },
    { "id": 2, "tasks": ["1.4", "2.1", "4.1"] },
    { "id": 3, "tasks": ["2.2", "2.3", "4.2"] },
    { "id": 4, "tasks": ["2.4", "2.5", "4.3", "5.1"] },
    { "id": 5, "tasks": ["5.2"] },
    { "id": 6, "tasks": ["5.3", "5.4", "6.1"] },
    { "id": 7, "tasks": ["6.2", "6.3", "6.4"] },
    { "id": 8, "tasks": ["7.1", "7.2", "7.3"] },
    { "id": 9, "tasks": ["7.4", "7.5", "7.6", "7.7", "7.8"] },
    { "id": 10, "tasks": ["9.1"] },
    { "id": 11, "tasks": ["9.2", "10.1"] },
    { "id": 12, "tasks": ["10.2", "11.1"] },
    { "id": 13, "tasks": ["11.2"] },
    { "id": 14, "tasks": ["12.1"] },
    { "id": 15, "tasks": ["12.2"] }
  ]
}
```
