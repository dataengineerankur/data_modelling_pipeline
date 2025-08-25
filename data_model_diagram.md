# Zocdoc-Style Data Model Diagram

## Data Model Overview

This data model is designed for efficient BI queries and handles SCD Type 2 for user identity resolution, ensuring unique user-level spend calculations without double counting.

## Entity Relationship Diagram

```
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│   dim_user      │    │  user_identity_map  │    │   dim_session   │
│   (SCD Type 2)  │◄───┤   (SCD Type 2)      │    │                 │
└─────────────────┘    └──────────────────────┘    └─────────────────┘
         │                       │                        │
         │                       │                        │
         ▼                       ▼                        ▼
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│fact_appointment │    │   fact_payment       │    │  dim_provider   │
│                 │    │                      │    │                 │
└─────────────────┘    └──────────────────────┘    └─────────────────┘
         │                       │                        │
         │                       │                        │
         ▼                       ▼                        ▼
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│dim_insurance_   │    │    dim_date          │    │   dim_time      │
│plan             │    │                      │    │                 │
└─────────────────┘    └──────────────────────┘    └─────────────────┘
```

## Key Design Principles

### 1. SCD Type 2 Implementation
- **dim_user**: Tracks user attribute changes over time
- **user_identity_map**: Resolves multiple identity types to canonical_user_id
- **Valid_from/valid_to**: Temporal validity windows for all SCD2 tables

### 2. Deduplication Strategy
- **Sessions**: Latest record per session_id (ROW_NUMBER() OVER PARTITION)
- **Payments**: Net by appointment_id (refunds are negative)
- **Appointments**: Unique by appointment_id

### 3. Identity Resolution
- **Primary**: login_user_id → canonical_user_id
- **Secondary**: device_id → canonical_user_id (for anonymous users)
- **Fallback**: Anonymous users get unique canonical_user_id

### 4. Business Logic
- **Net Revenue**: SUM(payments.amount) per appointment (refunds negative)
- **User Spend**: Attribute to canonical_user_id via appointment linkage
- **Session Attribution**: Last touch before appointment booking

## Table Structure Details

### Fact Tables
- **fact_appointment**: One row per appointment with all dimensions
- **fact_payment**: One row per payment with netting logic
- **fact_session**: One row per deduplicated session

### Dimension Tables
- **dim_user**: SCD2 user attributes with temporal validity
- **dim_provider**: Provider information (type 1)
- **dim_insurance_plan**: Insurance plan details (type 1)
- **dim_session**: Session metadata and attribution
- **dim_date/dim_time**: Standard time dimensions

### Bridge Tables
- **user_identity_map**: SCD2 identity resolution mapping

## Data Flow

1. **Raw Data Ingestion** → Staging tables
2. **Deduplication** → Remove duplicates from sessions and facts
3. **Identity Resolution** → Map to canonical_user_id using SCD2
4. **Netting** → Calculate net payments per appointment
5. **Dimension Loading** → Load SCD2 dimensions with change detection
6. **Fact Loading** → Load facts with proper dimension keys
7. **Aggregation** → Create user-level spend metrics

## Key Metrics

- **User Total Spend**: SUM(net_payment_amount) per canonical_user_id
- **Completed Appointments**: COUNT by status = 'completed'
- **Conversion Rate**: Appointments / Sessions
- **Revenue per User**: Total spend / Unique users
- **Provider Performance**: Appointments and revenue by provider
