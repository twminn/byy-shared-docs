# Feature Request: Opportunity Creation in Landing Leads API

> **Priority:** MEDIUM  
> **Status:** REQUESTED  
> **Requested By:** Frontend Team  
> **Date:** December 11, 2025

---

## Summary

The `/api/v1/landing_leads` endpoint currently creates contacts but does not create opportunities in GHL pipelines. The frontend is sending `pipeline` and `stage` parameters that are being ignored.

---

## Current Behavior

**Request sent by frontend:**
```json
{
  "email": "user@example.com",
  "source": "newsletter_signup",
  "page": "gift",
  "pipeline": "Engaged Prospects",
  "stage": "Submitted Lead Form"
}
```

**Current result:**
- ✅ Contact created in GHL
- ❌ No opportunity created
- ❌ `pipeline` and `stage` params ignored (not in `permit` list)

---

## Requested Behavior

After creating/updating the contact, create an opportunity in the specified pipeline and stage.

**Expected result:**
- ✅ Contact created/updated in GHL
- ✅ Opportunity created in specified pipeline
- ✅ Contact added to specified stage

---

## Use Cases

### 1. Newsletter Signup (Gift Landing Page)
```json
{
  "email": "user@example.com",
  "source": "newsletter_signup",
  "page": "gift",
  "pipeline": "Engaged Prospects",
  "stage": "Submitted Lead Form"
}
```

### 2. Demo Request
```json
{
  "email": "user@example.com",
  "firstName": "John",
  "source": "demo_request",
  "page": "new",
  "pipeline": "Sales Pipeline",
  "stage": "Demo Scheduled"
}
```

### 3. Contact-Only (No Opportunity)
```json
{
  "email": "user@example.com",
  "source": "signup_landing",
  "page": "new"
}
```
If `pipeline` is not provided, skip opportunity creation (backward compatible).

---

## Suggested Implementation

### 1. Update Permitted Params

```ruby
def lead_params
  params.permit(:firstName, :lastName, :email, :phone, :source, :page, :campaign, :pipeline, :stage)
end
```

### 2. Add Opportunity Creation Logic

```ruby
def create
  # ... existing contact creation logic ...
  
  # Create opportunity if pipeline specified
  if lead_params[:pipeline].present?
    create_opportunity(contact_id, lead_params[:pipeline], lead_params[:stage])
  end
  
  # ... render response ...
end

private

def create_opportunity(contact_id, pipeline_name, stage_name)
  ghl_service = GoHighLevelService.new
  
  # Look up pipeline ID by name
  pipeline = ghl_service.find_pipeline_by_name(pipeline_name)
  return unless pipeline
  
  # Look up stage ID by name
  stage = pipeline['stages'].find { |s| s['name'] == stage_name }
  stage_id = stage&.dig('id') || pipeline['stages'].first['id']
  
  # Create opportunity
  ghl_service.create_opportunity(
    contact_id: contact_id,
    pipeline_id: pipeline['id'],
    stage_id: stage_id,
    name: "#{lead_params[:source] || 'Landing Page'} Lead",
    status: 'open'
  )
rescue StandardError => e
  Rails.logger.warn("Opportunity creation failed: #{e.message}")
  # Don't fail the request if opportunity creation fails
end
```

### 3. GHL Service Methods Needed

```ruby
# Find pipeline by name
def find_pipeline_by_name(name)
  pipelines = get_pipelines
  pipelines.find { |p| p['name'] == name }
end

# Create opportunity
def create_opportunity(contact_id:, pipeline_id:, stage_id:, name:, status: 'open')
  body = {
    contactId: contact_id,
    pipelineId: pipeline_id,
    pipelineStageId: stage_id,
    name: name,
    status: status,
    locationId: @location_id
  }
  
  response = self.class.post(
    "/opportunities/",
    body: body.to_json,
    headers: api_headers
  )
  
  handle_response(response)
  response.parsed_response
end
```

---

## Updated API Spec

### Request Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | ✅ Yes | Contact email address |
| `firstName` | string | No | First name |
| `lastName` | string | No | Last name |
| `phone` | string | No | Phone number |
| `source` | string | No | Lead source identifier |
| `page` | string | No | Landing page identifier |
| `campaign` | string | No | Campaign identifier |
| **`pipeline`** | string | No | **NEW:** GHL Pipeline name for opportunity |
| **`stage`** | string | No | **NEW:** Pipeline stage name |

### Response (with opportunity)

```json
{
  "success": true,
  "contact_id": "abc123xyz",
  "opportunity_id": "opp456def",
  "action": "created",
  "message": "Contact and opportunity created successfully"
}
```

---

## Testing

### Test Case 1: With Pipeline
```bash
curl -X POST https://bestyearyet.io/api/v1/landing_leads \
  -H "Content-Type: application/json" \
  -H "Origin: https://gift.bestyearyet.com" \
  -d '{
    "email": "test-opp@example.com",
    "source": "newsletter_signup",
    "page": "gift",
    "pipeline": "Engaged Prospects",
    "stage": "Submitted Lead Form"
  }'
```
**Expected:** Contact created + Opportunity in "Engaged Prospects" pipeline

### Test Case 2: Without Pipeline (backward compatible)
```bash
curl -X POST https://bestyearyet.io/api/v1/landing_leads \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test-contact@example.com",
    "source": "signup_landing",
    "page": "new"
  }'
```
**Expected:** Contact created only (no opportunity)

---

## GHL Pipeline Reference

| Pipeline Name | Common Stages | Use Case |
|--------------|---------------|----------|
| Engaged Prospects | Submitted Lead Form, Contacted, Qualified | Newsletter signups, content downloads |
| Sales Pipeline | New Lead, Demo Scheduled, Proposal Sent, Closed | Sales-qualified leads |

---

## Timeline

| Date | Action |
|------|--------|
| Dec 11, 2025 | Feature requested |
| TBD | Rails team reviews |
| TBD | Implementation |
| TBD | Frontend verification |

