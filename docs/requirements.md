
# CivicFix — Product Requirements Document

**Version:** 1.0  
**Status:** Draft  
**Project Type:** AWS Cloud Engineering Portfolio Project  
**Initial Region:** Niagara, Ontario, Canada

## 1. Product Overview

CivicFix is a cloud-native civic issue reporting platform that allows residents to report infrastructure and environmental problems in their communities, track their status, and view reported issues on an interactive map.

The platform also provides an administrative dashboard for reviewing reports and managing their resolution.

The initial release is a portfolio demonstration using synthetic data and simulated municipal administrators. It is not affiliated with a municipal government.

## 2. Problem Statement

Residents encounter problems such as potholes, damaged sidewalks, broken streetlights, illegal dumping, and fallen trees.

Reporting these issues may require navigating multiple municipal reporting systems. Residents may also have limited visibility into the status of their submissions.

CivicFix aims to provide a simple, centralized reporting and tracking experience.

## 3. Product Goals

- Allow residents to submit location-based issue reports.
- Provide an accessible interface for tracking reports.
- Display reported issues on an interactive map.
- Enable authorized administrators to manage reports.
- Demonstrate a secure, scalable, cloud-native AWS architecture.

## 4. Target Users

### Residents
People who want to report community infrastructure problems and monitor reported issues.

### Administrators
Authorized users who review reports, update statuses, and manage the reporting workflow.

## 5. MVP Features

### FR-01: Authentication
- Residents can register and log in.
- Users can log out.
- Authentication is handled by Amazon Cognito.
- Administrative endpoints require administrator authorization.

### FR-02: Submit a Report
Authenticated residents can submit:
- Issue category
- Description
- Location coordinates
- Optional photograph

### FR-03: View Reports
- Residents can view their own reports.
- Residents can view publicly visible community reports.
- Each report displays its current status.

### FR-04: Interactive Map
- Display publicly visible reports on a map.
- Allow users to select a location when reporting.
- Filter map reports by category and status.

### FR-05: Administrator Dashboard
- Administrators can view submitted reports.
- Administrators can change report statuses.
- Status changes are recorded for auditing.

### FR-06: Report Status Tracking
Supported statuses:
- SUBMITTED
- UNDER_REVIEW
- IN_PROGRESS
- RESOLVED
- REJECTED

## 6. Nonfunctional Requirements

### Security
- HTTPS for all application traffic.
- Cognito JWT authentication for protected endpoints.
- Role-based authorization.
- Least-privilege IAM permissions.
- Private S3 storage for uploaded photographs.
- Input validation and upload restrictions.

### Performance
- Target p95 API latency below 500 ms for tested endpoints, excluding image uploads.
- Paginated report retrieval.

### Reliability
- CloudWatch logging and monitoring.
- Defined error handling.
- Infrastructure deployment through Terraform.
- Documented recovery procedures.

### Maintainability
- TypeScript frontend.
- Python backend.
- Automated tests.
- GitHub Actions CI/CD.
- Documented API contracts.

### Accessibility
- Responsive layout.
- Keyboard-accessible interactions.
- Accessible form labels and error messages.
- WCAG 2.2 AA as a design target.

## 7. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js, TypeScript, Tailwind CSS |
| Backend | Python, AWS Lambda |
| API | Amazon API Gateway |
| Authentication | Amazon Cognito |
| Database | Amazon DynamoDB |
| Storage | Amazon S3 |
| Hosting | Amazon S3 and CloudFront |
| Infrastructure | Terraform |
| CI/CD | GitHub Actions |
| Monitoring | Amazon CloudWatch |
| Maps | Leaflet |
| Testing | Pytest, Playwright |

## 8. Core User Stories

**US-01:** As a resident, I want to create an account so I can submit and track reports.

**US-02:** As a resident, I want to submit a report with a description, category, location, and photo so that a community issue can be documented.

**US-03:** As a resident, I want to view my previous reports so that I can monitor their progress.

**US-04:** As a resident, I want to view reports on a map so that I can understand issues affecting my community.

**US-05:** As an administrator, I want to review reported issues so that I can manage submissions.

**US-06:** As an administrator, I want to update report statuses so that residents can see progress.

## 9. Out of Scope for MVP

- Official municipal government integrations
- Native iOS and Android applications
- AI-powered issue classification
- Automated municipal work-order dispatch
- SMS notifications
- Payment processing
- Public comments and voting

## 10. MVP Acceptance Criteria

The MVP is complete when:

1. An authenticated resident can submit a valid issue report.
2. An uploaded photo is stored securely in S3.
3. Reports are persisted in DynamoDB.
4. Residents can retrieve their previous reports.
5. Publicly visible reports appear on the community map.
6. Authorized administrators can update report statuses.
7. Unauthorized users cannot modify administrative data.
8. Status updates are recorded in an audit history.
9. Application infrastructure can be deployed using Terraform.
10. CI/CD checks and core automated tests pass.

## 11. Success Metrics

The project will measure:

- Successful report submission rate.
- API latency under a defined test workload.
- API error rate.
- Deployment success rate.
- Automated test coverage.
- Estimated monthly AWS operating costs.

All measured performance and reliability results will be documented after testing.

## 12. Future Enhancements

- Amazon Bedrock-powered issue classification.
- Duplicate report detection.
- Email status notifications.
- Community analytics dashboard.
- Geospatial search optimization.
- Integration with authorized municipal systems.

## 13. Project Milestones

| Week | Deliverable |
|---|---|
| 1 | Requirements, wireframes, architecture, repository |
| 2 | AWS infrastructure and backend APIs |
| 3 | Resident reporting interface |
| 4 | Admin dashboard and end-to-end workflow |
| 5 | CI/CD, security, monitoring, performance testing |
| 6 | Documentation, case study, portfolio launch |
