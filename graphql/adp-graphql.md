# ADP GraphQL Schema

## Overview

ADP (Automatic Data Processing) provides cloud-based human capital management (HCM) solutions covering payroll, benefits, talent, time, tax, and HR services. This conceptual GraphQL schema represents the core data domains exposed through ADP's REST APIs — primarily the Workers API and Payroll API — translated into a unified GraphQL surface.

ADP's developer platform (https://developers.adp.com/) is organized around discrete REST endpoints for workforce data, payroll processing, benefits administration, and time tracking. This schema models those domains as interconnected GraphQL types, enabling clients to fetch nested worker, payroll, and benefits data in a single query instead of chaining multiple REST calls.

## Data Domains

### Worker and Person Data

The central entity is `Worker`, representing an individual employed by or contracted with an organization. Workers have rich personal information modeled through `Person`, `LegalName`, `PreferredName`, `PersonAddress`, `PersonPhone`, `PersonEmail`, and `PersonalInfo`. Demographic data including `BirthDate`, `Gender`, `MaritalStatus`, `USCitizenshipStatus`, and `ImmigrationStatus` is scoped under personal information.

Worker identity and authentication within ADP's systems is tracked via `WorkerAssociation` and `WorkerStatus`, reflecting the lifecycle states (Active, Terminated, Leave, etc.) that workers move through over time.

### Employment and Position

Employment history is captured through `Employment`, `EmploymentHistory`, `EmploymentStatus`, and key date markers: `HireDate`, `OriginalHireDate`, `ReHireDate`, and `TerminationDate`. Organizational placement is modeled through `Position` and `PositionDetails`, with structural context from `JobTitle`, `BusinessUnit`, `Department`, `Division`, and `Location`.

Reporting relationships are expressed via `Supervisor` and `ReportsTo`, allowing org-chart traversal within the graph.

### Payroll

Payroll data covers the full processing cycle. `PayrollGroup` and `PayGroup` define how workers are grouped for pay runs. `PayFrequency` and `PayPeriod` set the schedule. Individual pay results are captured in `PayStatement`, which breaks down into `Earning` (base pay, overtime, bonuses), `Deduction` (pre-tax and post-tax), `Tax` (federal, state, local withholding), and `NetPay`.

Wage and hour compliance is supported through `WageHour`, `TimeEntry`, and `TimeCard`. Direct deposit configuration is modeled via `DirectDeposit` and `BankAccount`.

### Time and Leave

`TimeOff` represents individual leave requests and approvals. `LeaveBalance` tracks accrued and remaining leave by type (vacation, sick, PTO, FMLA, etc.).

### Benefits

Benefits enrollment flows through `Benefits`, `BenefitEnrollment`, `BenefitPlan`, and `BenefitCoverage`. These types represent what plans are available, what workers are enrolled in, and at what coverage tier (employee only, employee + spouse, family, etc.).

### Security and Integration

`APIKey` and `OAuthToken` model ADP's authentication artifacts (ADP uses OAuth 2.0 client credentials flow for server-to-server integrations). `Webhook` models event subscriptions for real-time notification of worker lifecycle, payroll completion, and benefits enrollment events.

`WorkerImage` supports profile photo retrieval. `WorkerMessage` models in-platform notifications and HR communications.

## Schema Source

This is a conceptual schema derived from:
- ADP Developer Portal: https://developers.adp.com/
- ADP GitHub organization: https://github.com/adplabs
- ADP REST API documentation for Workers API, Payroll API, Benefits API, and Time API

ADP does not currently publish an official GraphQL endpoint. This schema represents how the REST API surface would be modeled in GraphQL, suitable for use in a GraphQL gateway or federation layer over ADP's REST APIs.

## Query Examples

```graphql
# Fetch a worker with current position and recent pay statement
query GetWorker($associateOID: ID!) {
  worker(associateOID: $associateOID) {
    associateOID
    workerStatus {
      statusCode
      effectiveDate
    }
    person {
      legalName {
        formattedName
        givenName
        familyName
      }
      personalInfo {
        birthDate { dateValue }
        gender { genderCode }
        maritalStatus { maritalStatusCode }
      }
    }
    position {
      jobTitle { titleName }
      department { departmentName }
      location { locationName }
    }
    payStatements(limit: 1) {
      payPeriod { startDate endDate }
      netPay { amount currencyCode }
      earnings { earningTypeName amount }
      taxes { taxTypeName amount jurisdictionCode }
    }
    leaveBalances {
      leaveTypeName balanceHours balanceDays
    }
    benefitEnrollments {
      benefitPlan { planName planType }
      coverageTier
      effectiveDate
    }
  }
}

# List workers in a department with pay group info
query ListDepartmentWorkers($departmentCode: String!) {
  workers(filter: { departmentCode: $departmentCode }) {
    totalCount
    workers {
      associateOID
      person { legalName { formattedName } }
      workerStatus { statusCode }
      payGroup { payGroupCode payGroupName }
      position { jobTitle { titleName } supervisor { formattedName } }
    }
  }
}

# Retrieve a pay statement detail
query GetPayStatement($associateOID: ID!, $payPeriodEndDate: String!) {
  payStatement(associateOID: $associateOID, payPeriodEndDate: $payPeriodEndDate) {
    payPeriod { startDate endDate }
    earnings { earningTypeName earningTypeCode amount hours rate }
    deductions { deductionTypeName preTax amount }
    taxes { taxTypeName jurisdictionCode amount }
    directDeposits { bankAccount { bankName accountType } amount }
    netPay { amount currencyCode }
  }
}
```

## Mutation Examples

```graphql
# Update a worker's contact information
mutation UpdateWorkerContact($input: UpdatePersonContactInput!) {
  updatePersonContact(input: $input) {
    worker {
      associateOID
      person {
        contactMethods {
          ... on PersonEmail { emailAddress isPrimary }
          ... on PersonPhone { phoneNumber phoneType }
        }
      }
    }
  }
}

# Enroll a worker in a benefit plan
mutation EnrollInBenefitPlan($input: BenefitEnrollmentInput!) {
  enrollInBenefitPlan(input: $input) {
    benefitEnrollment {
      benefitPlan { planName planType }
      coverageTier
      effectiveDate
      enrollmentStatus
    }
  }
}

# Register a webhook for worker lifecycle events
mutation RegisterWebhook($input: WebhookInput!) {
  registerWebhook(input: $input) {
    webhook {
      webhookID
      eventType
      targetURL
      isActive
    }
  }
}
```
