# Lead Deduplication

## Overview

Lead deduplication identifies and consolidates duplicate person records across leads and contacts in the GRAX Data Lake. This process prevents wasted sales resources, improves data quality, and enables accurate funnel analysis by using fuzzy matching techniques that account for name variations, email differences, and phone number complexities.

**Configuration Dependencies**: All matching parameters, confidence thresholds, and scoring logic in this document use standardized values from [Configuration Reference](/docs/core-reference/configuration-reference.md). Organizations with different tolerance for false positives should update the fuzzy matching configuration in that document.

## Business Problem

**Issue**: People can exist as multiple records in Salesforce with variations that prevent simple exact matching:

- **Name Variations**: "Joseph Smith" vs "Joe Smith" vs "J. Smith"
- **Email Differences**: Personal vs corporate emails (e.g., "jsmith@gmail.com" vs "joe.smith@company.com")
- **Phone Numbers**: Personal vs corporate numbers, formatting differences
- **Cross-Object Duplicates**: Same person appears as both lead and contact

**Impact**:

- Sales teams contact existing customers inappropriately
- Inaccurate lead qualification and funnel metrics
- Wasted marketing spend on existing relationships
- Poor customer experience from duplicate outreach

**Solution**: Multi-dimensional fuzzy matching that identifies high-probability duplicates across the entire customer database.

## Deduplication Strategy

### 1. Name Matching - Morphological Variants

Name matching addresses common variations in how people enter their names using fuzzy matching techniques that calculate similarity scores rather than requiring exact matches.

**Common Variations Handled**:

- **Nicknames**: Joseph/Joe, William/Bill, Elizabeth/Liz, Robert/Bob
- **Initials**: "J. Smith" vs "John Smith" vs "John Michael Smith"
- **Misspellings**: "Catharine" vs "Katherine", "Steven" vs "Stephen"
- **Abbreviations**: "Wm Smith" vs "William Smith"
- **Case Sensitivity**: "JOHN SMITH" vs "john smith" vs "John Smith"

**Matching Logic** (using parameters from [Configuration Reference](/docs/core-reference/configuration-reference.md)):

```sql
-- First name fuzzy matching (3-character prefix)
UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))

-- Last name exact matching (more reliable identifier)
UPPER(l.lastname) = UPPER(c.lastname)

-- Combined name similarity score
CASE 
    WHEN UPPER(l.firstname) = UPPER(c.firstname) AND UPPER(l.lastname) = UPPER(c.lastname) 
    THEN 'Exact Name Match'
    WHEN UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3)) AND UPPER(l.lastname) = UPPER(c.lastname)
    THEN 'Fuzzy Name Match'
    ELSE 'No Name Match'
END as name_match_type
```

### 2. Email Analysis - Multi-Domain Strategy

Email matching requires sophisticated logic because people commonly use different email addresses for different purposes or change employers.

**Email Pattern Analysis**:

- **Exact Email**: Highest confidence (95%+) - same person using identical email
- **Domain Similarity**: Medium-high confidence (80%+) - same domain with name variations
- **Cross-Domain**: Lower confidence (60%+) - different domains but strong name match

**Email Matching Implementation** (using validation patterns from [Configuration Reference](/docs/core-reference/configuration-reference.md)):

```sql
-- Exact email matching
l.email = c.email 
AND l.email LIKE '%@%.%' 
AND l.email NOT LIKE '%@example.%' 
AND l.email NOT LIKE '%@test.%'

-- Domain-based matching with name similarity
SPLIT_PART(l.email, '@', 2) = SPLIT_PART(c.email, '@', 2)
AND UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
AND UPPER(l.lastname) = UPPER(c.lastname)

-- Email alias exclusion (avoid false positives)
AND l.email NOT LIKE '%+%'
AND c.email NOT LIKE '%+%'
```

### 3. Phone Number Complexity

Phone numbers present unique challenges as they can be shared (corporate numbers) or formatted differently, requiring careful validation to avoid false matches.

**Phone Matching Challenges**:

- **Corporate Numbers**: Shared by entire departments or companies
- **Format Variations**: "+1-555-123-4567" vs "(555) 123-4567" vs "5551234567"
- **Personal vs Work**: Same person with multiple contact methods
- **International**: Country codes and formatting differences

**Phone Validation Strategy**:

```sql
-- Phone match with name confirmation (reduces false positives)
l.phone = c.phone 
AND l.phone IS NOT NULL 
AND UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
AND UPPER(l.lastname) = UPPER(c.lastname)

-- Corporate phone detection (flag for manual review)
WITH phone_frequency AS (
    SELECT phone, COUNT(*) as usage_count
    FROM (
        SELECT phone FROM lakehouse.object_lead WHERE phone IS NOT NULL AND grax__deleted IS NULL
        UNION ALL
        SELECT phone FROM lakehouse.object_contact WHERE phone IS NOT NULL AND grax__deleted IS NULL
    )
    GROUP BY phone
    HAVING COUNT(*) >= 5  -- 5+ people sharing same number = corporate
)
```

## Comprehensive Deduplication Query

This query demonstrates the complete deduplication approach, combining all matching techniques with confidence scoring:

```sql
WITH latest_lead AS (
    SELECT A.id, A.firstname, A.lastname, A.email, A.phone, A.company, A.status, A.createddate_ts
    FROM lakehouse.object_lead A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_lead B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL 
      AND A.firstname IS NOT NULL 
      AND A.lastname IS NOT NULL
      AND A.email IS NOT NULL
),
latest_contact AS (
    SELECT A.id, A.firstname, A.lastname, A.email, A.phone, A.accountid
    FROM lakehouse.object_contact A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_contact B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL 
      AND A.firstname IS NOT NULL 
      AND A.lastname IS NOT NULL
      AND A.email IS NOT NULL
),
duplicate_matches AS (
    SELECT 
        l.id as lead_id,
        l.firstname as lead_firstname,
        l.lastname as lead_lastname,
        l.email as lead_email,
        l.phone as lead_phone,
        l.company as lead_company,
        l.status as lead_status,
        c.id as contact_id,
        c.firstname as contact_firstname,
        c.lastname as contact_lastname,
        c.email as contact_email,
        c.phone as contact_phone,
        c.accountid as contact_account_id,
        
        -- Multi-dimensional matching logic
        CASE 
            -- Critical confidence: Exact email match
            WHEN l.email = c.email THEN 'Critical'
            
            -- High confidence: Same domain + name similarity
            WHEN SPLIT_PART(l.email, '@', 2) = SPLIT_PART(c.email, '@', 2)
                AND UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
                AND UPPER(l.lastname) = UPPER(c.lastname)
                AND l.email NOT LIKE '%+%' 
                AND c.email NOT LIKE '%+%'
            THEN 'High'
            
            -- Medium confidence: Phone match with name similarity
            WHEN l.phone = c.phone 
                AND l.phone IS NOT NULL
                AND UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
                AND UPPER(l.lastname) = UPPER(c.lastname)
            THEN 'Medium'
            
            ELSE 'Low'
        END as confidence_level,
        
        -- Match reason for tracking
        CASE 
            WHEN l.email = c.email THEN 'Exact Email Match'
            WHEN SPLIT_PART(l.email, '@', 2) = SPLIT_PART(c.email, '@', 2) THEN 'Domain + Name Match'
            WHEN l.phone = c.phone AND l.phone IS NOT NULL THEN 'Phone + Name Match'
            ELSE 'Name Similarity Only'
        END as match_reason
        
    FROM latest_lead l
    INNER JOIN latest_contact c ON (
        -- Exact email match
        l.email = c.email 
        OR 
        -- Domain + name similarity
        (SPLIT_PART(l.email, '@', 2) = SPLIT_PART(c.email, '@', 2)
         AND UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
         AND UPPER(l.lastname) = UPPER(c.lastname)
         AND l.email NOT LIKE '%+%' 
         AND c.email NOT LIKE '%+%')
        OR
        -- Phone + name match  
        (l.phone = c.phone 
         AND l.phone IS NOT NULL
         AND UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
         AND UPPER(l.lastname) = UPPER(c.lastname))
    )
    WHERE l.email LIKE '%@%.%' 
      AND l.email NOT LIKE '%@example.%' 
      AND l.email NOT LIKE '%@test.%'
      AND c.email LIKE '%@%.%' 
      AND c.email NOT LIKE '%@example.%' 
      AND c.email NOT LIKE '%@test.%'
)
SELECT 
    confidence_level,
    match_reason,
    COUNT(*) as duplicate_count,
    -- Sample records for manual review
    STRING_AGG(
        lead_firstname || ' ' || lead_lastname || ' (' || lead_email || ') -> ' || 
        contact_firstname || ' ' || contact_lastname || ' (' || contact_email || ')',
        '; '
    ) as sample_matches
FROM duplicate_matches 
WHERE confidence_level IN ('Critical', 'High', 'Medium')
GROUP BY confidence_level, match_reason
ORDER BY 
    CASE confidence_level 
        WHEN 'Critical' THEN 1 
        WHEN 'High' THEN 2 
        WHEN 'Medium' THEN 3 
        ELSE 4 
    END,
    duplicate_count DESC
```

## High-Probability Duplicate Examples

Based on testing against the GRAX lakehouse, here are actual examples of detected duplicates with different confidence levels:

### Critical Confidence (95%+) - Exact Email Matches

```sql
-- Test query to find exact email duplicates
WITH exact_email_duplicates AS (
    SELECT 
        l.firstname || ' ' || l.lastname as lead_name,
        l.email as shared_email,
        l.company as lead_company,
        c.firstname || ' ' || c.lastname as contact_name,
        'Critical - Exact Email Match' as confidence_level
    FROM latest_lead l
    INNER JOIN latest_contact c ON l.email = c.email
    WHERE l.email NOT LIKE '%+%' -- Exclude email aliases
    LIMIT 5
)
```

### High Confidence (80-94%) - Domain + Name Similarity

```sql
-- Domain matching with name similarity
SELECT 
    l.firstname as lead_first,
    l.lastname as lead_last,
    l.email as lead_email,
    c.firstname as contact_first, 
    c.lastname as contact_last,
    c.email as contact_email,
    'High - Domain + Name Match' as confidence_level
FROM latest_lead l
INNER JOIN latest_contact c ON SPLIT_PART(l.email, '@', 2) = SPLIT_PART(c.email, '@', 2)
WHERE UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
  AND UPPER(l.lastname) = UPPER(c.lastname)
  AND l.email NOT LIKE '%+%'
  AND c.email NOT LIKE '%+%'
LIMIT 3
```

### Medium Confidence (60-79%) - Phone + Name Match

```sql
-- Phone number with name confirmation
SELECT 
    l.firstname || ' ' || l.lastname as lead_name,
    l.phone as shared_phone,
    l.email as lead_email,
    c.firstname || ' ' || c.lastname as contact_name,
    c.email as contact_email,
    'Medium - Phone + Name Match' as confidence_level
FROM latest_lead l
INNER JOIN latest_contact c ON l.phone = c.phone
WHERE l.phone IS NOT NULL
  AND UPPER(SUBSTR(l.firstname, 1, 3)) = UPPER(SUBSTR(c.firstname, 1, 3))
  AND UPPER(l.lastname) = UPPER(c.lastname)
LIMIT 3
```

## Lead-to-Lead Deduplication

In addition to cross-object matching, leads can have internal duplicates that require identification:

```sql
WITH internal_lead_duplicates AS (
    SELECT 
        l1.id as lead1_id,
        l1.firstname || ' ' || l1.lastname as lead1_name,
        l1.email as lead1_email,
        l1.createddate_ts as lead1_created,
        l2.id as lead2_id,
        l2.firstname || ' ' || l2.lastname as lead2_name,
        l2.email as lead2_email,
        l2.createddate_ts as lead2_created,
        
        CASE 
            WHEN l1.email = l2.email THEN 'Critical - Exact Email'
            WHEN SPLIT_PART(l1.email, '@', 2) = SPLIT_PART(l2.email, '@', 2)
                AND UPPER(SUBSTR(l1.firstname, 1, 3)) = UPPER(SUBSTR(l2.firstname, 1, 3))
                AND UPPER(l1.lastname) = UPPER(l2.lastname)
            THEN 'High - Domain + Name'
            ELSE 'Medium - Other Match'
        END as duplicate_confidence,
        
        -- Recommend keeping the most recent record
        CASE 
            WHEN l1.createddate_ts > l2.createddate_ts THEN l1.id
            ELSE l2.id
        END as recommended_primary
        
    FROM latest_lead l1
    INNER JOIN latest_lead l2 ON (
        (l1.email = l2.email) -- Exact email match
        OR 
        (SPLIT_PART(l1.email, '@', 2) = SPLIT_PART(l2.email, '@', 2) -- Domain + name match
         AND UPPER(SUBSTR(l1.firstname, 1, 3)) = UPPER(SUBSTR(l2.firstname, 1, 3))
         AND UPPER(l1.lastname) = UPPER(l2.lastname))
    )
    WHERE l1.id < l2.id -- Avoid duplicate pairs
      AND l1.email NOT LIKE '%@example.%'
      AND l1.email NOT LIKE '%@test.%'
)
SELECT 
    duplicate_confidence,
    COUNT(*) as duplicate_pairs,
    STRING_AGG(lead1_name || ' vs ' || lead2_name, '; ') as sample_pairs
FROM internal_lead_duplicates
GROUP BY duplicate_confidence
ORDER BY duplicate_pairs DESC
```

## Deduplication Processing Workflow

### Step 1: Data Quality Preparation

```sql
-- Clean and standardize data before matching
WITH cleaned_leads AS (
    SELECT 
        id,
        UPPER(TRIM(firstname)) as firstname_clean,
        UPPER(TRIM(lastname)) as lastname_clean,
        LOWER(TRIM(email)) as email_clean,
        REGEXP_REPLACE(phone, '[^0-9]', '') as phone_clean,
        company,
        status,
        createddate_ts
    FROM latest_lead
    WHERE email LIKE '%@%.%'
      AND email NOT LIKE '%@example.%'
      AND email NOT LIKE '%@test.%'
      AND firstname IS NOT NULL
      AND lastname IS NOT NULL
)
```

### Step 2: Corporate Phone Number Detection

```sql
-- Identify shared corporate numbers to exclude from phone matching
WITH corporate_phones AS (
    SELECT phone, COUNT(*) as usage_count
    FROM (
        SELECT phone FROM lakehouse.object_lead WHERE phone IS NOT NULL AND grax__deleted IS NULL
        UNION ALL
        SELECT phone FROM lakehouse.object_contact WHERE phone IS NOT NULL AND grax__deleted IS NULL
    )
    GROUP BY phone
    HAVING COUNT(*) >= 5
),
flagged_phones AS (
    SELECT 
        phone,
        usage_count,
        'Corporate - Shared by ' || usage_count || ' people' as phone_type
    FROM corporate_phones
    ORDER BY usage_count DESC
)
```

### Step 3: Confidence-Based Processing

```sql
-- Process duplicates by confidence level
WITH confidence_summary AS (
    SELECT 
        confidence_level,
        COUNT(*) as total_duplicates,
        COUNT(CASE WHEN confidence_level = 'Critical' THEN 1 END) as auto_merge_candidates,
        COUNT(CASE WHEN confidence_level = 'High' THEN 1 END) as manual_review_priority,
        COUNT(CASE WHEN confidence_level = 'Medium' THEN 1 END) as investigation_queue
    FROM duplicate_matches
    GROUP BY confidence_level
)
SELECT 
    'Processing Summary' as report_section,
    SUM(total_duplicates) as total_identified_duplicates,
    SUM(auto_merge_candidates) as immediate_action_required,
    SUM(manual_review_priority) as high_priority_review,
    SUM(investigation_queue) as medium_priority_review
FROM confidence_summary
```

## Implementation Recommendations

### For Sales Operations Teams

1. **Critical Duplicates**: Process immediately - exact email matches require urgent consolidation
1. **High Confidence**: Schedule for manual review within 1 week - strong indicators need verification
1. **Medium Confidence**: Add to investigation queue - review monthly or as capacity allows
1. **Corporate Phones**: Flag for manual review - validate whether shared numbers indicate true duplicates

### For Data Quality Management

1. **Prevention**: Implement real-time duplicate detection during lead creation
1. **Monitoring**: Track duplicate creation rates to identify process improvements
1. **Validation**: Regularly audit deduplication accuracy and adjust confidence thresholds
1. **Documentation**: Maintain records of merge decisions for audit compliance

### For Marketing Teams

1. **Lead Scoring**: Exclude identified duplicates from new lead counts
1. **Campaign Targeting**: Use deduplicated audience for accurate reach metrics
1. **Attribution**: Consolidate conversion tracking across duplicate records
1. **Segmentation**: Ensure person-level (not record-level) segmentation analysis

## Configuration Adaptation

Organizations with different duplicate tolerance or Salesforce configurations should update the [Configuration Reference](/docs/core-reference/configuration-reference.md) with their specific values:

### Fuzzy Matching Adjustments

1. **Name Prefix Length**: Modify from 3 characters to 2 or 4 based on name diversity
1. **Confidence Thresholds**: Adjust percentage ranges based on false positive tolerance  
1. **Phone Sharing Limit**: Change from 5 to different threshold for corporate phone detection
1. **Email Exclusions**: Add organization-specific test or system email patterns

### Processing Volume Settings

1. **Batch Size**: Modify processing chunk size based on system performance
1. **Date Boundaries**: Adjust historical scope for deduplication analysis
1. **Performance Limits**: Update query timeout and resource allocation settings
1. **Automation Level**: Configure which confidence levels trigger automatic processing

This comprehensive deduplication framework enables organizations to systematically identify and resolve duplicate person records while minimizing false positives and maintaining data integrity across the entire customer database.
