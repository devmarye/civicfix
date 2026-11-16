
# CivicFix — UI/UX Design Specification

**Version:** 1.0  
**Status:** Draft  
**Project:** CivicFix  
**Last Updated:** November 16, 2026

## 1. Design Overview

CivicFix is a responsive, mobile-first civic issue reporting platform that enables residents to document community problems, explore reported issues, and monitor their progress.

The interface should be simple, accessible, trustworthy, and efficient for both residents and administrators.

### Design Principles

1. **Simplicity:** Minimize the number of steps required to report an issue.
2. **Accessibility:** Design for users with different abilities, devices, and technical experience.
3. **Transparency:** Make report statuses and history easy to understand.
4. **Consistency:** Reuse visual components and interaction patterns.
5. **Mobile-first:** Prioritize residents reporting issues from smartphones.
6. **Trust:** Clearly communicate privacy, data usage, and the demonstration status of the platform.

## 2. Brand Identity

### Product Name

CivicFix

### Tagline

Spot it. Report it. Fix it.

### Brand Personality

- Approachable
- Reliable
- Community-focused
- Modern
- Professional
- Transparent

### Color Palette

| Token | Hex | Usage |
|---|---|---|
| Primary | #123B60 | Headings, navigation, primary brand |
| Secondary | #159C86 | Main actions, links, success accents |
| Background | #F4F7FB | Page backgrounds |
| Surface | #FFFFFF | Cards, forms, navigation |
| Accent | #F5A623 | Highlights and warnings |
| Text Primary | #1F2937 | Body text |
| Text Secondary | #52667A | Supporting text |
| Error | #DC2626 | Errors and destructive actions |
| Border | #D8E1EB | Dividers and input outlines |

Status indicators must use text labels as well as color.

### Typography

**Primary font:** Inter

Recommended hierarchy:

| Element | Font Size | Weight |
|---|---|---|
| Hero Heading | 40–48px desktop / 30–36px mobile | 700 |
| Page Heading | 28–32px | 700 |
| Section Heading | 20–24px | 600 |
| Card Heading | 16–18px | 600 |
| Body Text | 16px | 400 |
| Supporting Text | 14px | 400 |
| Button Text | 14–16px | 600 |

### Layout

- Mobile-first responsive design
- Maximum content width: 1200px
- Spacing scale: 4, 8, 12, 16, 24, 32, 48px
- Card border radius: 12px
- Button border radius: 8px
- Minimum interactive target: 24 × 24 CSS pixels, with larger targets preferred
- Consistent spacing between related elements
- Clearly visible keyboard focus states

## 3. Information Architecture

### Public Routes

| Route | Page |
|---|---|
| `/` | Homepage |
| `/map` | Community issue map |
| `/reports/[id]` | Public report details |
| `/login` | Sign in |
| `/register` | Create account |

### Authenticated Resident Routes

| Route | Page |
|---|---|
| `/report` | Submit issue |
| `/dashboard` | My reports |

### Administrator Routes

| Route | Page |
|---|---|
| `/admin` | Admin overview |
| `/admin/reports` | Report management |
| `/admin/reports/[id]` | Administrative report details |

Backend APIs must enforce authorization independently of frontend route protection.

## 4. Screen Specifications

### 4.1 Homepage

**Purpose:** Introduce CivicFix and guide visitors toward reporting or exploring issues.

#### Components

- Logo and navigation
- Hero section
- Primary CTA: Report an Issue
- Secondary CTA: Explore Issues
- How It Works section
- Issue category overview
- Footer with privacy information and demo disclaimer

#### Hero Content

**Heading:** See a problem? Make a difference.

**Description:** Report potholes, broken streetlights, illegal dumping, and other neighborhood issues in just a few steps.

**Primary CTA:** Report an Issue

**Secondary CTA:** Explore Issues

#### User Interactions

- Report an Issue opens `/report`.
- Explore Issues opens `/map`.
- Sign In opens `/login`.
- On mobile, navigation collapses into an accessible menu.

### 4.2 Authentication

**Purpose:** Allow residents and administrators to access protected functionality.

#### Components

- Email field
- Password field
- Sign-in button
- Registration link
- Form validation
- Loading and error states

#### Requirements

- Authentication through Amazon Cognito.
- Passwords must not be stored by CivicFix.
- Users receive clear authentication error messages.
- Successful login redirects to the appropriate page.
- Administrative access is based on authorized roles, not user-selected roles.

### 4.3 Report an Issue

**Purpose:** Enable residents to submit geotagged issue reports.

#### Reporting Workflow

**Step 1 — Issue details**

- Select issue category.
- Enter description.
- Optionally upload a photograph.

**Step 2 — Location**

- Select a location on an interactive map.
- Display selected coordinates or approximate address.
- Allow the resident to reposition the location marker.

**Step 3 — Review and submit**

- Display entered details.
- Allow edits before submission.
- Submit report.
- Display confirmation and tracking ID.

#### Categories

- Pothole / Road Damage
- Sidewalk Damage
- Broken Streetlight
- Illegal Dumping
- Fallen Tree
- Other

#### Validation

- Category is required.
- Description is required.
- Description must be between 10 and 1,000 characters.
- Valid geographic coordinates are required.
- One optional image is supported in the MVP.
- Accepted image types: JPEG, PNG, and WebP.
- Maximum upload size: 5 MB.
- Validate input and upload restrictions on the backend.

#### UI States

- Initial
- Validation error
- Uploading image
- Submitting report
- Submission successful
- Submission failed

#### Success Message

"Your report has been submitted successfully."

Display the report tracking ID and a link to view the report.

### 4.4 Community Map

**Purpose:** Allow visitors to explore community issues.

#### Components

- Interactive map
- Issue markers
- Category filter
- Status filter
- Report list
- Report preview cards
- Empty-state message

#### Interactions

- Selecting a marker displays an issue summary.
- Selecting an issue opens its detail page.
- Filters update the map and report list.
- Map queries use geographic bounds and pagination or result limits.
- Only publicly visible, non-sensitive report information is displayed.

#### Responsive Behavior

**Desktop:** Map and issue list displayed side by side.

**Mobile:** Map displayed above the issue list, with accessible filter controls.

The map must have a usable text-based alternative for keyboard and screen-reader users.

### 4.5 My Reports Dashboard

**Purpose:** Enable residents to track their submissions.

#### Components

- Page heading
- Report summary
- Status badges
- Report cards or table
- Creation dates
- Links to report details
- Empty state

#### Example Empty State

"You haven't submitted any reports yet."

CTA: Report an Issue

### 4.6 Report Details

**Purpose:** Display information and status history for an individual issue.

#### Components

- Report category
- Description
- Report photograph, if available
- Approximate public location
- Current status
- Submission date
- Status history timeline
- Tracking ID

Private owner details must not be exposed on public pages.

### 4.7 Admin Dashboard

**Purpose:** Enable authorized administrators to manage submitted reports.

#### Components

- Summary statistics
- Report management table
- Category and status filters
- Report detail panel
- Status update controls
- Status history

#### Status Workflow

SUBMITTED → UNDER_REVIEW → IN_PROGRESS → RESOLVED

A report may also be marked REJECTED by an authorized administrator.

The backend defines and validates allowed status transitions.

#### Administrator Actions

- View reports
- Filter reports
- Open report details
- Change report status
- Provide a status update reason
- Review status history

All status updates must be recorded with timestamps and the acting administrator's identifier.

## 5. Shared UI Components

Develop reusable components for:

- Header
- Navigation
- Footer
- Button
- Text input
- Textarea
- Select
- File upload
- Status badge
- Issue card
- Report list
- Map marker
- Modal / confirmation dialog
- Loading spinner
- Skeleton loader
- Toast notification
- Empty state
- Error message
- Pagination controls

Components should support consistent styling and keyboard interactions.

## 6. Responsive Design

### Breakpoints

| Device | Width |
|---|---|
| Mobile | Below 640px |
| Tablet | 640–1023px |
| Desktop | 1024px and above |

### Mobile Requirements

- Single-column forms
- Responsive navigation
- Readable text without horizontal scrolling
- Touch-friendly buttons
- Accessible file uploads
- Responsive maps
- Clearly visible primary actions

### Desktop Requirements

- Maximum content width of 1200px
- Wider forms and dashboard layouts
- Side-by-side map and report list
- Responsive report tables
- Clear navigation and hierarchy

## 7. Accessibility

Target WCAG 2.2 AA conformance.

Requirements include:

- Semantic HTML
- Keyboard-accessible navigation
- Visible focus indicators
- Sufficient color contrast
- Labels for every form control
- Accessible validation messages
- Text alternatives for meaningful images
- Accessible dialog and menu interactions
- Status information communicated without relying on color
- Reduced-motion support where relevant

Automated accessibility checks should be supplemented with manual testing.

## 8. Privacy and Safety

The initial launch uses synthetic demonstration reports.

For any future real-user deployment:

- Explain how submitted information will be used.
- Avoid displaying reporter identity publicly.
- Avoid exposing sensitive location details.
- Store uploaded photographs in private S3 buckets.
- Provide a process for reporting inappropriate content.
- Define data retention and deletion policies.
- Avoid publishing unreviewed reports containing personal information.

CivicFix must clearly identify itself as an independent demonstration platform, not an official municipal service.

## 9. Loading, Empty, and Error States

Every data-driven screen should support:

**Loading:** Display appropriate skeletons or progress indicators.

**Empty:** Explain when no reports exist or match the selected filters.

**Error:** Explain what failed and offer retry options where appropriate.

**Success:** Confirm completed actions without ambiguity.

Avoid exposing internal API errors or stack traces to end users.

## 10. Design Deliverables

The UI/UX phase is complete when:

- [ ] Brand colors and typography are defined.
- [ ] Homepage wireframe is completed.
- [ ] Authentication screens are designed.
- [ ] Reporting workflow is designed.
- [ ] Community map is designed.
- [ ] My Reports dashboard is designed.
- [ ] Report details screen is designed.
- [ ] Admin dashboard is designed.
- [ ] Mobile and desktop layouts are considered.
- [ ] Accessibility requirements are documented.
- [ ] Loading, empty, success, and error states are defined.

## 11. Future UX Enhancements

- AI-assisted report categorization
- Duplicate issue suggestions
- Multilingual support
- Email notifications
- Community analytics
- More advanced geospatial search
- Authorized municipal integrations

## 12. Design Implementation

**Frontend framework:** Next.js  
**Language:** TypeScript  
**Styling:** Tailwind CSS  
**Maps:** Leaflet  
**Design tool:** Figma (optional)

The frontend implementation should follow this specification while allowing changes based on usability testing and engineering constraints.
