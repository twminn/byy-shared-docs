# CORS Configuration Required for Landing Pages API

> **Priority:** HIGH  
> **Status:** ✅ COMPLETED  
> **Affects:** All landing page lead capture forms  
> **Date:** December 11, 2025

---

## 🚨 Issue Summary

Landing pages cannot submit leads to the Rails API due to missing CORS (Cross-Origin Resource Sharing) configuration. Browser security blocks cross-origin requests from landing page domains to `bestyearyet.io`.

### Error Observed
```
TypeError: Failed to fetch
```

Browser blocks the request because the API doesn't respond to CORS preflight (OPTIONS) requests.

---

## 📋 Technical Details

### What's Happening

1. User fills out form on `gift.bestyearyet.com`
2. JavaScript attempts `POST https://bestyearyet.io/api/v1/landing_leads`
3. Browser first sends OPTIONS preflight request to check CORS policy
4. Rails returns 404 (no CORS headers)
5. Browser blocks the actual POST request
6. Lead is not captured

### Domains Requiring CORS Access

| Domain | Purpose |
|--------|---------|
| `https://gift.bestyearyet.com` | Gift subscription landing page |
| `https://new.bestyearyet.com` | Main signup landing page |
| `https://bestyearyet.com` | Primary marketing website |
| `https://www.bestyearyet.com` | WWW variant |
| `http://localhost:3000` | Local development |
| `http://127.0.0.1:3000` | Local development |

### API Endpoint Affected

```
POST /api/v1/landing_leads
```

---

## ✅ Required Action (Rails Team)

### Step 1: Add rack-cors gem

```ruby
# Gemfile
gem 'rack-cors'
```

```bash
bundle install
```

### Step 2: Create CORS initializer

Create `config/initializers/cors.rb`:

```ruby
# config/initializers/cors.rb
#
# CORS configuration for landing page API endpoints
# 
# This allows our landing pages (hosted on different subdomains)
# to make API requests to the Rails app for lead capture.
#
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    # Landing page domains
    origins 'gift.bestyearyet.com',
            'new.bestyearyet.com',
            'bestyearyet.com',
            'www.bestyearyet.com'
    
    # Landing leads endpoint only
    resource '/api/v1/landing_leads',
      headers: :any,
      methods: [:post, :options],
      credentials: false,
      max_age: 86400
  end
  
  # Development/staging environments
  if Rails.env.development? || Rails.env.staging?
    allow do
      origins 'localhost:3000',
              '127.0.0.1:3000',
              /\Ahttp:\/\/localhost:\d+\z/
      
      resource '/api/v1/landing_leads',
        headers: :any,
        methods: [:post, :options],
        credentials: false
    end
  end
end
```

### Step 3: Restart Rails server

```bash
# Development
rails restart

# Production (depends on your setup)
# Heroku: git push heroku main
# AWS: deployment pipeline will handle
```

---

## 🧪 Testing Instructions

### Test 1: Verify OPTIONS preflight works

```bash
curl -X OPTIONS https://bestyearyet.io/api/v1/landing_leads \
  -H "Origin: https://gift.bestyearyet.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  -v
```

**Expected Response Headers:**
```
Access-Control-Allow-Origin: https://gift.bestyearyet.com
Access-Control-Allow-Methods: POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```

### Test 2: Verify POST works with Origin header

```bash
curl -X POST https://bestyearyet.io/api/v1/landing_leads \
  -H "Origin: https://gift.bestyearyet.com" \
  -H "Content-Type: application/json" \
  -d '{"email":"cors-test@example.com","source":"cors_test"}' \
  -v
```

**Expected:** 201 Created with `Access-Control-Allow-Origin` header

### Test 3: Browser test

1. Go to https://gift.bestyearyet.com
2. Scroll to newsletter signup section
3. Enter email and submit
4. Check browser console - should show success, no CORS errors
5. Verify contact created in GHL

---

## 📊 Request/Response Format

### Request
```http
POST /api/v1/landing_leads HTTP/1.1
Host: bestyearyet.io
Origin: https://gift.bestyearyet.com
Content-Type: application/json

{
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "source": "newsletter_signup",
  "page": "gift",
  "pipeline": "Engaged Prospects",
  "stage": "Submitted Lead Form"
}
```

### Response (Success)
```http
HTTP/1.1 201 Created
Access-Control-Allow-Origin: https://gift.bestyearyet.com
Content-Type: application/json

{
  "success": true,
  "contact_id": "abc123xyz",
  "action": "created",
  "message": "Contact created successfully"
}
```

---

## 🔒 Security Notes

1. **Specific Origins**: Only allow known landing page domains, not `*` wildcard
2. **Limited Endpoints**: Only expose `/api/v1/landing_leads`, not all API routes
3. **Limited Methods**: Only POST and OPTIONS, not GET/PUT/DELETE
4. **No Credentials**: Set `credentials: false` since we don't need cookies
5. **Cache Preflight**: `max_age: 86400` caches preflight for 24 hours

---

## 📞 Contact

**Frontend Team:** Questions about landing page integration  
**Rails Team:** API implementation questions

---

## 📅 Timeline

| Date | Action |
|------|--------|
| Dec 11, 2025 | Issue identified, documentation created |
| Dec 11, 2025 | ✅ Rails team implemented CORS configuration |
| Dec 11, 2025 | ✅ Tested on staging - OPTIONS preflight returns 200 with correct headers |
| TBD | Frontend team verifies fix on landing pages |

### Implementation Notes

**Commit:** `fix(cors): Update CORS config for landing page API access`

**Changes made to `config/initializers/cors.rb`:**
- Added explicit HTTPS protocols to origin URLs
- Separated production and development/staging origins  
- Added all required landing page domains
- Reference: [byy repo commit](https://github.com/twminn/byy)

**Test Results (Staging):**
```
OPTIONS /api/v1/landing_leads → 200 OK
  access-control-allow-origin: https://gift.bestyearyet.com ✅
  access-control-allow-methods: POST, OPTIONS ✅
  access-control-allow-headers: Content-Type ✅
  access-control-max-age: 86400 ✅
```

