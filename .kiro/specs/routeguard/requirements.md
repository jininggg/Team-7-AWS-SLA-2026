# Requirements Document

## Introduction

RouteGuard is a mobile application for the AWS-SLA Geospatial Innovation Challenge, addressing challenge statement (b): real-time accessibility disruption alerts for persons with disabilities. RouteGuard acts as an orchestration layer that integrates OneMap's Barrier-Free Access (BFA) routing with AWS services to deliver three core capabilities: Disruption Alerts, Risk Scores, and Stranded Mode. The MVP prioritizes Disruption Alert as the first fully working feature, followed by Risk Score, then Stranded Mode.

## Glossary

- **RouteGuard**: The mobile application orchestration layer that coordinates disruption detection, alerting, and rerouting for persons with disabilities
- **BFA_Route**: A Barrier-Free Access route retrieved from OneMap's BFA Routing API, representing a wheelchair-accessible path between two points
- **Disruption**: An accessibility obstacle event (e.g., lift breakdown, blocked ramp, construction on accessible path) that renders part of a BFA_Route impassable
- **Disruption_Ingestion_Service**: The AWS Lambda function responsible for receiving and validating disruption reports from data sources
- **Route_Matcher**: The AWS Lambda function that compares active disruptions against users' saved BFA_Routes to identify affected users
- **Alert_Service**: The component responsible for generating and delivering disruption notifications to affected users via AWS SNS
- **Risk_Score_Engine**: The AWS Lambda function that calculates historical disruption frequency per location using a weighted count stored in DynamoDB
- **Stranded_Mode_Service**: The AWS Lambda function that handles SOS requests by generating reroutes, notifying caregivers, and surfacing nearby accessible help points
- **User_Profile**: A DynamoDB record containing a user's saved BFA_Routes, notification preferences, and designated caregiver contacts
- **Caregiver**: A designated contact who receives notifications when a user activates Stranded Mode
- **Help_Point**: A nearby accessible facility or service sourced from OneMap's "Programmes and Services for Persons with Disabilities" thematic layer
- **OneMap_API**: The set of OneMap APIs used by RouteGuard, including BFA Routing, Routing, Geocoding, and Search APIs
- **Alert_Message_Generator**: The AWS Bedrock (Claude) component that converts structured disruption data into plain-language alert messages
- **Location_Node**: A specific point along a BFA_Route (e.g., a lift, ramp, or crossing) that can be individually tracked for disruption history

## Requirements

### Requirement 1: Disruption Data Ingestion

**User Story:** As a system administrator, I want RouteGuard to ingest accessibility disruption data from external sources, so that the system has up-to-date information about obstacles on BFA routes.

#### Acceptance Criteria

1. WHEN a disruption report is received, THE Disruption_Ingestion_Service SHALL validate that the report contains a location (latitude within 1.15 to 1.47, longitude within 103.60 to 104.05 for Singapore), a disruption type from the enumerated set (lift_breakdown, escalator_breakdown, blocked_ramp, blocked_path, construction, flooding, other), and a timestamp in ISO 8601 format
2. IF a disruption report fails validation, THEN THE Disruption_Ingestion_Service SHALL reject the report with an HTTP 400 response and return an error message indicating which fields are missing or invalid
3. WHEN a valid disruption report is received, THE Disruption_Ingestion_Service SHALL store the disruption record in DynamoDB with a status of "active"
4. WHEN a disruption resolution report is received, THE Disruption_Ingestion_Service SHALL update the corresponding disruption record status to "resolved" and record the resolution timestamp
5. IF a disruption resolution report references a disruption ID that does not exist in DynamoDB, THEN THE Disruption_Ingestion_Service SHALL reject the report with an HTTP 404 response
6. THE Disruption_Ingestion_Service SHALL associate each disruption with a geospatial coordinate that can be matched against BFA_Route segments
7. THE Disruption_Ingestion_Service SHALL expose a source-agnostic ingestion interface (accepting a standard disruption payload schema) so that any data source (sample seed data, LTA feed, SMRT lift-status reports, or future live feeds) can submit disruptions without requiring changes to the service
8. WHEN deployed for the MVP prototype, THE Disruption_Ingestion_Service SHALL be seeded with at least 10 sample disruption records modeled on real LTA and SMRT lift-status report formats to demonstrate end-to-end functionality
9. IF DynamoDB storage fails when persisting a disruption record, THEN THE Disruption_Ingestion_Service SHALL return an HTTP 500 response and log the failure details for operational monitoring

### Requirement 2: Route Matching

**User Story:** As a person with a disability, I want RouteGuard to detect when a disruption affects my saved route, so that I am warned before encountering an impassable obstacle.

#### Acceptance Criteria

1. WHEN a new active disruption is stored, THE Route_Matcher SHALL identify all User_Profiles with saved BFA_Routes that pass within 50 metres of the disruption location
2. WHEN a match is found between a disruption and a BFA_Route, THE Route_Matcher SHALL trigger the Alert_Service for each affected user, passing the disruption details (type, location, timestamp) and the affected route identifier
3. WHEN a new active disruption is stored, THE Route_Matcher SHALL complete matching for all affected users within 30 seconds of the disruption being stored
4. WHEN a user saves a new BFA_Route, THE Route_Matcher SHALL check the route against all currently active disruptions and alert the user if any exist
5. IF the Route_Matcher encounters an error during the matching process, THEN THE Route_Matcher SHALL log the error and retry the matching operation once before marking the disruption as unmatched for manual review
6. WHEN a disruption status changes to "resolved", THE Route_Matcher SHALL cease matching that disruption against any BFA_Routes

### Requirement 3: Pre-Departure and En-Route Alerts

**User Story:** As a person with a disability, I want to receive timely alerts about disruptions on my route, so that I can plan an alternative before leaving or before reaching the affected point.

#### Acceptance Criteria

1. WHEN the Route_Matcher identifies an affected user who has not yet departed, THE Alert_Service SHALL send a pre-departure alert via AWS SNS push notification within 60 seconds of the match being identified
2. WHILE a user is en-route on a saved BFA_Route, WHEN a new disruption appears at least 200 metres ahead of the user's current position, THE Alert_Service SHALL send an en-route alert via AWS SNS push notification within 60 seconds
3. WHEN an alert is triggered, THE Alert_Message_Generator SHALL produce a plain-language message describing the disruption type, affected location, and whether the user's route is fully blocked or partially obstructed
4. THE Alert_Service SHALL include a deep link to the map view centered on the disruption coordinates in every alert notification
5. IF the Alert_Message_Generator fails to produce a message within 5 seconds, THEN THE Alert_Service SHALL fall back to a structured template-based message containing disruption type and location
6. THE Alert_Service SHALL send at most one alert per disruption per user per BFA_Route, unless the disruption status changes
7. THE Alert_Service SHALL determine departure status based on the user's last known location: a user is considered "not yet departed" if their location is within 100 metres of the route origin, and "en-route" if their location is along the route beyond 100 metres from the origin

### Requirement 4: User Route Management

**User Story:** As a person with a disability, I want to save and manage my frequently used BFA routes, so that RouteGuard can monitor them for disruptions.

#### Acceptance Criteria

1. WHEN a user requests a BFA route between an origin and destination, THE RouteGuard SHALL call the OneMap_API BFA Routing endpoint and return the route to the user
2. IF the OneMap_API BFA Routing endpoint returns an error or is unavailable, THEN THE RouteGuard SHALL display an error message to the user indicating that routing is temporarily unavailable
3. WHEN a user saves a BFA_Route, THE RouteGuard SHALL store the route geometry (as an ordered array of coordinate pairs), origin address, destination address, route name, and creation timestamp in the User_Profile in DynamoDB
4. THE RouteGuard SHALL allow a user to save up to 10 BFA_Routes in their User_Profile
5. IF a user attempts to save a BFA_Route when 10 routes are already saved, THEN THE RouteGuard SHALL reject the save and inform the user they have reached the maximum route limit
6. WHEN a user deletes a saved BFA_Route, THE RouteGuard SHALL remove the route from the User_Profile and cease monitoring that route for disruptions
7. WHEN a user requests a route, THE RouteGuard SHALL use OneMap_API Geocoding to resolve address inputs to coordinates before calling BFA Routing
8. IF the OneMap_API Geocoding returns multiple results for an address input, THEN THE RouteGuard SHALL present the top 5 results to the user for selection before proceeding with routing
9. IF the OneMap_API Geocoding returns zero results for an address input, THEN THE RouteGuard SHALL inform the user that the address could not be found and prompt them to refine the input

### Requirement 5: Risk Score Calculation

**User Story:** As a person with a disability, I want to see a reliability risk score for locations on my route, so that I can make informed decisions about which routes to trust.

#### Acceptance Criteria

1. WHEN a disruption is resolved, THE Risk_Score_Engine SHALL increment the disruption count for the affected Location_Node in DynamoDB and record the resolution timestamp
2. THE Risk_Score_Engine SHALL calculate a risk score as an integer from 0 to 100 for each Location_Node using a time-weighted count where disruptions within the past 30 days each carry a weight of 2 and disruptions from 31 to 90 days ago each carry a weight of 1, with disruptions older than 90 days excluded from the calculation
3. WHEN a user views a saved BFA_Route, THE RouteGuard SHALL display the numeric risk score (0-100) for each Location_Node along the route
4. WHEN the daily recalculation schedule triggers, THE Risk_Score_Engine SHALL recalculate risk scores for all Location_Nodes to exclude disruptions older than 90 days
5. WHEN a Location_Node has zero disruptions in the past 90 days, THE Risk_Score_Engine SHALL assign a risk score of zero
6. IF the Risk_Score_Engine fails to calculate a risk score for a Location_Node, THEN THE RouteGuard SHALL display an indicator that the risk score is unavailable for that location

### Requirement 6: Stranded Mode SOS

**User Story:** As a person with a disability who is stranded due to an unexpected obstacle, I want a one-tap SOS function, so that I can quickly get help and an alternative route.

#### Acceptance Criteria

1. WHEN a user activates Stranded Mode, THE Stranded_Mode_Service SHALL request a new BFA_Route from OneMap_API using the user's current location as the origin and original destination as the endpoint
2. IF no active route exists when Stranded Mode is activated, THEN THE Stranded_Mode_Service SHALL skip rerouting and proceed with caregiver notification and Help_Point lookup using the user's current location only
3. WHEN a user activates Stranded Mode, THE Stranded_Mode_Service SHALL send a notification to all designated Caregivers in the User_Profile via AWS SNS containing the user's current location (as a map link), a timestamp, and the message "Your contact has activated SOS and needs assistance"
4. WHEN a user activates Stranded Mode, THE Stranded_Mode_Service SHALL query OneMap_API's "Programmes and Services for Persons with Disabilities" thematic layer for Help_Points within 500 metres of the user's current location and display up to the 5 nearest results
5. THE Stranded_Mode_Service SHALL present the reroute, caregiver notification status, and nearby Help_Points to the user within 10 seconds of activation
6. IF the OneMap_API BFA reroute request fails, THEN THE Stranded_Mode_Service SHALL display the nearest Help_Points and caregiver notification status without a reroute
7. IF no designated Caregiver is configured in the User_Profile, THEN THE Stranded_Mode_Service SHALL skip caregiver notification and inform the user that no caregiver is set
8. IF the user's current location cannot be determined, THEN THE Stranded_Mode_Service SHALL display an error message prompting the user to enable location services
9. IF zero Help_Points are found within 500 metres, THEN THE Stranded_Mode_Service SHALL expand the search radius to 1000 metres and inform the user that no nearby help was found within 500 metres

### Requirement 7: User Authentication and Profile

**User Story:** As a person with a disability, I want a secure account so that my routes, preferences, and caregiver contacts are stored safely.

#### Acceptance Criteria

1. THE RouteGuard SHALL authenticate users via AWS Cognito before granting access to route management, alerts, or Stranded Mode features
2. IF authentication fails or a user's session has expired, THEN THE RouteGuard SHALL deny access to protected features and redirect the user to the login screen
3. WHEN a user registers, THE RouteGuard SHALL create a User_Profile in DynamoDB linked to the Cognito identity
4. WHEN a user updates caregiver contact information in their User_Profile, THE RouteGuard SHALL validate that the caregiver contact contains a phone number in E.164 international format or an email address conforming to standard mailbox format (local-part@domain)
5. IF caregiver contact validation fails, THEN THE RouteGuard SHALL reject the update and display an error message indicating which field is invalid
6. THE RouteGuard SHALL allow a user to designate up to 3 Caregivers in their User_Profile
7. IF a user attempts to add a Caregiver when 3 Caregivers are already designated, THEN THE RouteGuard SHALL reject the addition and display an error message indicating the maximum caregiver limit has been reached

### Requirement 8: Plain-Language Alert Generation

**User Story:** As a person with a disability, I want alerts written in simple, clear language, so that I can quickly understand what is wrong and what to do.

#### Acceptance Criteria

1. WHEN the Alert_Message_Generator receives disruption data, THE Alert_Message_Generator SHALL produce a message at a maximum reading level of Grade 6 (Flesch-Kincaid)
2. WHEN generating a message, THE Alert_Message_Generator SHALL include the following in every generated message: what happened (a plain-language description of the disruption), where it happened (the affected route, stop, or area), and what the user should do next (a specific actionable step)
3. WHEN generating a message, THE Alert_Message_Generator SHALL complete generation within 5 seconds measured from the moment disruption data is received to the moment the message is available for delivery
4. THE Alert_Message_Generator SHALL produce messages of 450 characters or fewer, including spaces, to accommodate the three required elements at Grade 6 reading level within mobile notification constraints
5. THE Alert_Message_Generator SHALL NOT use abbreviations, acronyms, or transit-specific jargon unless the term is immediately followed by a plain-language explanation within the same message
6. IF the Alert_Message_Generator receives disruption data that is missing one or more of the three required elements (what happened, where it happened, or what to do next), THEN THE Alert_Message_Generator SHALL generate the message using available data and indicate which information is currently unavailable
