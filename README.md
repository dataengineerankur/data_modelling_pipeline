# # Zocdoc-Style Data Model Implementation

## Overview

This project implements a comprehensive data model for a Zocdoc-style healthcare appointment booking platform. The model is designed to handle complex user identity resolution, SCD Type 2 tracking, and provide efficient BI queries for user-level spend analysis without double counting.

## Key Features

- **SCD Type 2 Implementation**: Tracks user identity changes over time
- **Identity Resolution**: Maps multiple identity types (login_user_id, device_id) to canonical_user_id
- **Payment Netting**: Handles refunds and chargebacks correctly
- **Session Attribution**: Links sessions to appointments for conversion analysis
- **RFM Analysis**: Recency, Frequency, Monetary customer segmentation
- **Data Quality Checks**: Comprehensive validation and integrity checks

## Data Model Architecture

### Source Data Structure

The model processes the following source files:

1. **sessions_csv.csv**: User browsing sessions with channel attribution
2. **user_identity_map_csv.csv**: SCD2 mapping for identity resolution
3. **appointments_csv.csv**: Healthcare appointments with status tracking
4. **payments_csv.csv**: Payment transactions including refunds/chargebacks
5. **providers_csv.csv**: Healthcare provider information
6. **insurance_plans_csv.csv**: Insurance plan details

### Data Model Layers

```
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│   Raw Data      │    │   Staging Layer      │    │   Data Model    │
│   (CSV Files)   │───▶│   (Cleaned Data)     │───▶│   (Star Schema) │
└─────────────────┘    └──────────────────────┘    └─────────────────┘
```

### Star Schema Design

- **Fact Tables**: fact_appointment, fact_payment, fact_session
- **Dimension Tables**: dim_user (SCD2), dim_provider, dim_insurance_plan, dim_session
- **Bridge Tables**: user_identity_map (SCD2)
- **Time Dimensions**: dim_date, dim_time

## Implementation Files

### 1. Data Model Diagram (`data_model_diagram.md`)
- Entity relationship diagram
- Design principles and patterns
- Data flow explanation

### 2. SQL Implementation (`sql_implementation.sql`)
- Complete table creation scripts
- Staging layer implementation
- SCD2 logic implementation
- Data quality checks
- Performance indexes

### 3. Final Tableau Query (`final_tableau_query.sql`)
- Comprehensive BI query
- All key metrics and dimensions
- RFM analysis and segmentation
- Ready for Tableau dashboard

## Key Business Logic

### User Identity Resolution

```sql
-- Resolve to canonical user using SCD2
COALESCE(
  u_login.canonical_user_id,    -- Primary: login_user_id
  u_device.canonical_user_id,   -- Secondary: device_id
  'ANON-' || device_id          -- Fallback: anonymous
) AS canonical_user_id
```

### Payment Netting

```sql
-- Handle refunds and chargebacks
CASE 
  WHEN payment_status IN ('refunded', 'chargeback') THEN -ABS(amount)
  ELSE ABS(amount)
END AS net_amount
```

### Session Deduplication

```sql
-- Keep latest session record
ROW_NUMBER() OVER (
  PARTITION BY session_id
  ORDER BY updated_at DESC
) = 1
```

## SCD Type 2 Implementation

### Key Components

1. **Valid_from/valid_to**: Temporal validity windows
2. **is_current**: Flag for current version
3. **hashdiff**: Change detection hash
4. **Point-in-time joins**: Historical analysis support

### Example SCD2 Logic

```sql
-- Update existing records when changes detected
WHEN MATCHED AND d.hashdiff <> s.hashdiff THEN
  UPDATE SET 
    d.valid_to = s.effective_ts,
    d.is_current = FALSE

-- Insert new versions
WHEN NOT MATCHED THEN
  INSERT (valid_from, valid_to, is_current)
  VALUES (s.effective_ts, '9999-12-31', TRUE)
```

## Key Metrics and KPIs

### User-Level Metrics
- **Total Spend**: Net payment amount per user
- **Appointment Completion Rate**: Completed vs. total appointments
- **Session Conversion Rate**: Sessions to appointments ratio
- **Revenue per User**: Average spend per user

### Business Metrics
- **Provider Performance**: Appointments and revenue by provider
- **Channel Attribution**: Conversion rates by marketing channel
- **Insurance Plan Analysis**: Revenue by plan type
- **Daily Revenue Trends**: Time-based revenue analysis

### RFM Segmentation
- **Champions**: High recency, frequency, and monetary scores
- **Loyal Customers**: Consistent users with good spending
- **At Risk**: Declining engagement users
- **Can't Lose**: High-value users at risk
- **Lost**: Inactive users

## Data Quality Framework

### Validation Checks

1. **SCD2 Validity**: No overlapping time windows
2. **Referential Integrity**: All foreign keys exist
3. **Payment Netting**: Gross vs. net amount validation
4. **Uniqueness**: No duplicate business keys
5. **Completeness**: Required fields not null

### Monitoring Views

- `v_scd2_validity_check`: SCD2 overlap detection
- `v_referential_integrity_check`: Orphaned record detection
- `v_payment_netting_check`: Payment validation

## Performance Optimization

### Indexing Strategy

```sql
-- Key performance indexes
CREATE INDEX idx_fact_appointment_user_sk ON fact_appointment(user_sk);
CREATE INDEX idx_dim_user_canonical_id ON dim_user(canonical_user_id);
CREATE INDEX idx_dim_user_valid_from ON dim_user(valid_from);
CREATE INDEX idx_dim_user_valid_to ON dim_user(valid_to);
```

### Query Optimization

- **Partitioning**: Date-based partitioning for time-series data
- **Clustering**: User-based clustering for user-centric queries
- **Materialized Views**: Pre-aggregated metrics for common queries

## Usage Instructions

### 1. Setup and Installation

```bash
# Navigate to project directory
cd project2_cdc_delta/data_modelling_pipeline

# Review source data
ls -la src/
```

### 2. Database Setup

```sql
-- Execute the complete implementation
\i sql_implementation.sql

-- Verify table creation
\dt
```

### 3. Data Loading

```sql
-- Load sample data (adjust paths as needed)
COPY raw_sessions FROM 'src/sessions_csv.csv' CSV HEADER;
COPY raw_user_identity_map FROM 'src/user_identity_map_csv.csv' CSV HEADER;
-- ... repeat for other tables
```

### 4. Tableau Integration

```sql
-- Use the final query for Tableau
\i final_tableau_query.sql

-- The view provides all metrics needed for dashboards
SELECT * FROM v_user_spend_summary LIMIT 100;
```

## Dashboard Recommendations

### 1. User Overview Dashboard
- User count by segment
- Total revenue and spend trends
- Conversion funnel metrics

### 2. Provider Performance Dashboard
- Provider revenue ranking
- Specialty performance analysis
- Network status impact

### 3. Marketing Attribution Dashboard
- Channel conversion rates
- Campaign performance
- Session to appointment funnel

### 4. Financial Dashboard
- Daily/monthly revenue trends
- Payment method analysis
- Refund and chargeback tracking

## Data Pipeline Considerations

### Incremental Processing

- **Change Detection**: Use hashdiff for SCD2 updates
- **Idempotency**: Handle duplicate data gracefully
- **Late Arriving Data**: Support for historical updates

### Monitoring and Alerting

- **Data Quality**: Automated validation checks
- **Performance**: Query execution time monitoring
- **Volume**: Record count trend analysis

## Troubleshooting

### Common Issues

1. **SCD2 Overlaps**: Check validity window logic
2. **Identity Resolution**: Verify mapping table integrity
3. **Payment Netting**: Validate refund handling
4. **Performance**: Review index usage and query plans

### Debug Queries

```sql
-- Check SCD2 validity
SELECT * FROM v_scd2_validity_check;

-- Verify referential integrity
SELECT * FROM v_referential_integrity_check;

-- Validate payment netting
SELECT * FROM v_payment_netting_check;
```

## Future Enhancements

### Potential Improvements

1. **Real-time Processing**: Stream processing for live dashboards
2. **Machine Learning**: Predictive analytics for user behavior
3. **Advanced Attribution**: Multi-touch attribution models
4. **Data Lineage**: End-to-end data flow tracking

### Scalability Considerations

- **Partitioning**: Time-based data partitioning
- **Caching**: Redis for frequently accessed metrics
- **Compression**: Columnar storage optimization
- **Distribution**: Multi-node cluster support

## Support and Maintenance

### Regular Tasks

- **Data Quality Monitoring**: Daily validation checks
- **Performance Tuning**: Query optimization and index maintenance
- **Backup and Recovery**: Regular data backups
- **Documentation Updates**: Keep implementation docs current

### Contact Information

For questions or support regarding this data model implementation, please refer to the project documentation or contact the data engineering team.

---

**Note**: This implementation is designed for educational and demonstration purposes. For production use, additional security, performance, and compliance considerations should be addressed.
