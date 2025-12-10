# GHL Private Integration Migration Plan

**Date**: December 6-7, 2025  
**Status**: COMPLETED  
**Priority**: Medium (no sunset date announced yet)

---

## Executive Summary

Go High Level transitioned from API v1.0 (traditional API Keys) to Private Integration Tokens (PITs) using API v2.0. While no sunset date has been announced, API v1.0 is at end-of-support with no maintenance. This document outlines the migration that was completed for BYY's GHL integration.

**Migration Completed**: December 7, 2025

---

## Migration Results

### Summary
- **D2C Pipeline**: Fully migrated and tested - 168 users synced successfully
- **Gift Pipeline**: Fully migrated and configured - ready for new gift subscriptions
- **All Users Synced**: Including workshop/gift code users who were previously excluded

### Key Metrics
- Users processed: 168
- Users updated: 84 
- Users skipped (already current): 84
- Errors: 0

---

## Current State (Post-Migration)

| Aspect | Value |
|--------|-------|
| **Base URL** | `https://services.leadconnectorhq.com` |
| **Authentication** | Bearer token (PIT) |
| **Token Format** | `pit-{uuid}` format |
| **Version Header** | `Version: 2021-07-28` |
| **Location ID** | `nXTbTZime6k6jGP5iVrz` |

---

## Environment Variables (AWS Secrets Manager)

### D2C Pipeline (Year 1)
| Variable | Purpose |
|----------|---------|
| `GHL_ACCESS_TOKEN` | Private Integration Token |
| `GHL_LOCATION_ID` | GHL Location ID |
| `GHL_ENABLED` | Enable/disable GHL integration |
| `GHL_PIPELINE_D2C_YEAR1_ID` | D2C Year 1 Pipeline ID |
| `GHL_STAGE_EMAIL_REG_EXPERIENCE_ID` | Registration stage |
| `GHL_STAGE_PAYMENT_ID` | Payment/Subscription stage |
| `GHL_STAGE_PLAN_COMPLETE_ID` | Plan complete stage |
| `GHL_STAGE_PLAN_MONTH1_ID` - `GHL_STAGE_PLAN_MONTH12_ID` | Monthly stages |

### Gift Pipeline (Purchaser Journey)
| Variable | Purpose |
|----------|---------|
| `GHL_PIPELINE_GIFT_ID` | Gift Purchaser Pipeline ID |
| `GHL_PIPELINE_GIFTS_ID` | Alias for gift pipeline |
| `GHL_STAGE_GIFT_PURCHASED_ID` | Gift purchased stage |
| `GHL_STAGE_GIFT_PENDING_ID` | Gift pending stage |
| `GHL_STAGE_GIFT_PARTIALLY_REDEEMED_ID` | Partially redeemed |
| `GHL_STAGE_GIFT_FULLY_REDEEMED_ID` | Fully redeemed |
| `GHL_STAGE_GIFT_PRE_RENEWAL_ID` | Pre-renewal stage |
| `GHL_STAGE_GIFT_CANCELLED_ID` | Cancelled stage |

### Gift Pipeline (Recipient Journey)
| Variable | Purpose |
|----------|---------|
| `GHL_PIPELINE_GIFT_RECIPIENT_ID` | Gift Recipient Pipeline ID |
| `GHL_STAGE_GIFT_SENT_ID` | Gift sent stage |
| `GHL_STAGE_GIFT_EMAIL_OPENED_ID` | Email opened stage |
| `GHL_STAGE_GIFT_EMAIL_CLICKED_ID` | Email clicked stage |
| `GHL_STAGE_GIFT_RECIPIENT_REGISTERED_ID` | Recipient registered |

### Gift Tags
| Variable | Purpose |
|----------|---------|
| `GHL_TAG_GIFT_RECIPIENT` | Tag for gift recipients |
| `GHL_TAG_GIFT_SPONSOR_PREFIX` | Prefix for sponsor tags |
| `GHL_TAG_GIFT_COHORT_PREFIX` | Prefix for cohort tags |

---

## Critical API v2 Differences Discovered

### 1. Contact Updates - NO locationId
**v1 Behavior**: `locationId` could be included in PUT requests
**v2 Behavior**: `locationId` MUST NOT be included in PUT requests (422 error)

```ruby
# WRONG - Will cause 422 error
self.class.put("/contacts/#{contact_id}", body: { locationId: @location_id, ... })

# CORRECT
self.class.put("/contacts/#{contact_id}", body: { firstName: "...", ... })
```

### 2. Contact Creation - locationId Required
**v2 Behavior**: `locationId` MUST be included in POST requests

```ruby
# CORRECT for creation
self.class.post("/contacts/", body: { locationId: @location_id, ... })
```

### 3. Custom Fields - Different Format
**v1 Format**: `customField: { field_name: "value" }`
**v2 Format**: `customFields: [{ id: "field_id", value: "value" }]`

**Solution**: We removed custom fields from payloads entirely as they weren't essential for our use case.

### 4. Opportunity Stage Updates
**v1 Field**: `stageId`
**v2 Field**: `pipelineStageId`

```ruby
# WRONG
{ stageId: new_stage_id }

# CORRECT
{ pipelineStageId: new_stage_id }
```

### 5. Opportunity Name Field
**v1 Field**: `title`
**v2 Field**: `name`

```ruby
# WRONG
{ title: "User Opportunity" }

# CORRECT  
{ name: "User Opportunity" }
```

### 6. Contact Search Endpoint
**v1**: `GET /contacts/?email=user@example.com`
**v2**: `GET /contacts/?query=user@example.com&locationId=xxx`

```ruby
# CORRECT v2 search
self.class.get("/contacts/", query: { 
  query: email, 
  locationId: @location_id 
})
```

### 7. Opportunities Search
**v1**: `GET /pipelines/{pipeline_id}/opportunities`
**v2**: `GET /opportunities/search?location_id=xxx&contact_id=xxx&pipeline_id=xxx`

---

## Files Changed

### Core Service Files
| File | Changes Made |
|------|--------------|
| `app/services/go_high_level_service.rb` | Updated base_uri, all endpoint paths, field names, added `create_contact_raw` and `update_contact_raw` methods |
| `app/services/ghl_pipeline_service.rb` | Updated base_uri, endpoint paths, field names |
| `app/services/gift_subscription_service.rb` | Updated to use raw contact methods, fixed field names |
| `app/models/gift_subscription.rb` | Updated `stageId` to `pipelineStageId` |

### Model Changes
| File | Changes Made |
|------|--------------|
| `app/models/user.rb` | Removed workshop user exclusion, added `bypasses_payment?` method |
| `app/models/concerns/ghl_stage_tracking.rb` | Removed workshop user exclusion |

### Rake Tasks Created
| File | Purpose |
|------|---------|
| `lib/tasks/ghl_workshop_migration.rake` | Full GHL sync tasks for all users |

### Tasks Available
```bash
# Audit all users (read-only)
rails ghl:audit_all

# Dry run - show what would change
rails ghl:audit_dry_run

# Full sync - update all users
rails ghl:sync_all_users
```

---

## AWS Secrets Manager - Important Notes

### JSON Format Requirements
AWS Secrets Manager requires **valid JSON**. Common issues:

1. **All keys must be quoted**: `"KEY": "value"` not `KEY: value`
2. **No trailing commas**: Last item should NOT have comma
3. **No BOM characters**: Use ASCII encoding when updating via file
4. **No hidden characters**: PowerShell can introduce encoding issues

### Safe Update Process
```powershell
# 1. Get current value
aws secretsmanager get-secret-value --profile byy --secret-id staging-byy-apikeys --query SecretString --output text > secrets.json

# 2. Edit with care (ensure valid JSON)

# 3. Write with ASCII encoding
[System.IO.File]::WriteAllText("$PWD\secrets-clean.json", $cleanJson, [System.Text.Encoding]::ASCII)

# 4. Update secret
aws secretsmanager update-secret --profile byy --secret-id staging-byy-apikeys --secret-string file://secrets-clean.json --region us-east-1

# 5. Force ECS deployment to pick up changes
aws ecs update-service --profile byy --cluster staging --service staging-best-year-yet --force-new-deployment --region us-east-1
```

---

## Compatibility Testing Checklist (COMPLETED)

### Contact Operations
- [x] Search contacts by email
- [x] Create new contact
- [x] Update existing contact
- [x] Add tags to contact

### Opportunity Operations
- [x] List opportunities by contact + pipeline
- [x] Create new opportunity
- [x] Update opportunity stage
- [x] Move opportunity to next month stage

### User Registration Flow
- [x] New user registration creates GHL contact
- [x] Payment completion updates opportunity to paid stage
- [x] Login updates stage based on last page
- [x] Workshop/gift code users placed directly in paid stage

### Gift Pipeline
- [x] Gift purchaser pipeline accessible
- [x] Gift recipient pipeline accessible
- [x] All gift stage IDs configured
- [x] Gift subscription service updated for v2

### Rake Tasks
- [x] `ghl:audit_all` works without GHL secrets (read-only)
- [x] `ghl:sync_all_users` syncs all users correctly
- [x] No backward stage movements

---

## Monitoring

### CloudWatch Log Patterns
```
# Success patterns
"GHL: Creating contact for user"
"GHL: Contact created successfully"
"GHL: Opportunity created"
"Moving opportunity"
"Forward:" or "Skipping (already current)"

# Error patterns (investigate immediately)
"GHL: Authentication failed"
"GHL: 401 Unauthorized"
"Client error: 422"
"Client error: 403"
```

---

## Known Limitations

1. **Sidekiq Not Deployed**: GHL calls are synchronous. If async is needed, deploy Sidekiq ECS service.

2. **Custom Fields Not Supported**: v2 API uses different format. Currently not sending custom fields.

3. **Task Definition Updates**: CI/CD may not always apply Terraform changes. Manual task definition registration may be needed for new secrets.

---

## Rollback Plan

If issues occur:

### Option 1: Revert Token
1. Update `GHL_ACCESS_TOKEN` in AWS Secrets Manager to old v1 token
2. Revert code changes in `go_high_level_service.rb` to use v1 base_uri
3. Restart ECS services

### Option 2: Disable GHL
1. Set `GHL_ENABLED=false` in AWS Secrets Manager
2. Restart ECS services
3. GHL integration disabled but app continues working

---

## References

- [GHL Private Integrations Guide](https://help.gohighlevel.com/support/solutions/articles/155000003054-private-integrations-everything-you-need-to-know)
- [GHL API v2 Documentation](https://highlevel.stoplight.io/)
- [LeadConnector API Reference](https://services.leadconnectorhq.com)

---

## Migration Timeline

| Date | Action |
|------|--------|
| Dec 6, 2025 | Planning document created |
| Dec 7, 2025 | Local environment updated with PIT |
| Dec 7, 2025 | Services updated for v2 API |
| Dec 7, 2025 | Staging secrets updated |
| Dec 7, 2025 | First deployment - 422 errors discovered |
| Dec 7, 2025 | Fixed locationId and customField issues |
| Dec 7, 2025 | Full sync completed - 168 users processed |
| Dec 7, 2025 | Gift pipeline verified and configured |
| Dec 7, 2025 | **Migration Complete** |

---

**Status**: MIGRATION COMPLETE - All systems operational
