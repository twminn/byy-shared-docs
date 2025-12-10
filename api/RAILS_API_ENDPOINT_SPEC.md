# Rails API Endpoint Specification for Landing Page Leads

## Overview

This document provides the complete implementation spec for adding a lead capture API endpoint to the BYY Rails application. This endpoint will be called by the landing pages (new.bestyearyet.com, gift.bestyearyet.com, etc.) to create/update contacts in GoHighLevel CRM.

**Target Endpoint:** `POST /api/v1/landing_leads`  
**Authentication:** None (public endpoint with rate limiting)  
**CORS:** Allow requests from `*.bestyearyet.com`

---

## ✅ Implementation Status

**Implemented:** December 10, 2025  
**Release:** v3.0.50 (pending)

### Endpoint URLs
- **Staging:** `https://byy-staging.bestyearyet.io/api/v1/landing_leads`
- **Production:** `https://bestyearyet.io/api/v1/landing_leads`

### Implementation Notes for Frontend Team

The Rails implementation uses existing, production-tested GHL service methods. Key differences from the original spec:

| Original Spec | Actual Implementation | Notes |
|---------------|----------------------|-------|
| `rack-attack` rate limiting | Custom Redis-backed rate limiting | Same limits: 10/min IP, 3/hr email |
| Single `update_contact` call | `add_tags_to_contact` + `update_contact_raw` | Preserves existing contact tags |
| Generic error message | Same generic message | GHL errors not exposed to client |

### Files Created
- `app/controllers/api/v1/public_base_controller.rb` - Base controller with rate limiting
- `app/controllers/api/v1/landing_leads_controller.rb` - Lead capture endpoint
- `config/initializers/cors.rb` - CORS configuration for BYY domains
- `spec/requests/api/v1/landing_leads_spec.rb` - Full test coverage

### CORS Allowed Origins
- `bestyearyet.com` and `www.bestyearyet.com`
- Any `*.bestyearyet.com` subdomain (regex pattern)
- `localhost:3000`, `localhost:3111`, `localhost:8000` (development)

---

## Implementation

### 1. Routes Configuration

Add to `config/routes.rb`:

```ruby
namespace :api do
  namespace :v1 do
    resources :landing_leads, only: [:create]
  end
end
```

---

### 2. Controller Implementation

Create `app/controllers/api/v1/landing_leads_controller.rb`:

```ruby
# frozen_string_literal: true

module Api
  module V1
    class LandingLeadsController < ApplicationController
      # Skip authentication - this is a public endpoint
      skip_before_action :authenticate_user!, raise: false
      skip_before_action :verify_authenticity_token
      
      # Rate limiting (if using rack-attack or similar)
      # throttle requests to prevent abuse
      
      # CORS preflight
      def options
        head :ok
      end

      def create
        # Validate required fields
        unless valid_email?(lead_params[:email])
          return render json: { 
            success: false, 
            error: 'Valid email is required' 
          }, status: :unprocessable_entity
        end

        # Prepare contact data
        contact_data = build_contact_data(lead_params)

        begin
          # Use existing GoHighLevelService
          ghl_service = GoHighLevelService.new
          
          # Search for existing contact
          existing_contact = ghl_service.search_contact_by_email(contact_data[:email])
          
          if existing_contact
            # Update existing contact (add tags, update info)
            result = ghl_service.update_contact(existing_contact['id'], contact_data)
            render json: {
              success: true,
              contact_id: existing_contact['id'],
              action: 'updated',
              message: 'Contact updated successfully'
            }, status: :ok
          else
            # Create new contact
            result = ghl_service.create_contact(contact_data)
            contact_id = result.dig('contact', 'id') || result['id']
            render json: {
              success: true,
              contact_id: contact_id,
              action: 'created',
              message: 'Contact created successfully'
            }, status: :created
          end

        rescue StandardError => e
          Rails.logger.error("Landing lead creation failed: #{e.message}")
          Rails.logger.error(e.backtrace.first(10).join("\n"))
          
          render json: {
            success: false,
            error: 'Failed to process lead. Please try again.'
          }, status: :internal_server_error
        end
      end

      private

      def lead_params
        params.permit(:firstName, :lastName, :email, :phone, :source, :page, :campaign)
      end

      def valid_email?(email)
        email.present? && email.match?(/\A[^@\s]+@[^@\s]+\.[^@\s]+\z/)
      end

      def build_contact_data(params)
        {
          firstName: params[:firstName].to_s.strip,
          lastName: params[:lastName].to_s.strip,
          email: params[:email].to_s.downcase.strip,
          phone: params[:phone].to_s.strip,
          source: params[:source] || 'landing_page',
          tags: build_tags(params)
        }
      end

      def build_tags(params)
        tags = ['landing_page_lead']
        
        # Add page-specific tag
        case params[:page]
        when 'new'
          tags << 'new_bestyearyet_com'
        when 'gift'
          tags << 'gift_bestyearyet_com'
        when 'main'
          tags << 'bestyearyet_com'
        else
          tags << 'landing_page'
        end
        
        # Add source-specific tag
        tags << "source_#{params[:source]}" if params[:source].present?
        
        # Add campaign tag if provided
        tags << "campaign_#{params[:campaign]}" if params[:campaign].present?
        
        tags.uniq
      end
    end
  end
end
```

---

### 3. CORS Configuration

If not already configured, add CORS support. Using the `rack-cors` gem:

Add to `Gemfile` (if not present):
```ruby
gem 'rack-cors'
```

Add/update `config/initializers/cors.rb`:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    # Allow requests from BYY landing pages
    origins 'new.bestyearyet.com', 
            'gift.bestyearyet.com', 
            'bestyearyet.com',
            'www.bestyearyet.com',
            /\Ahttps:\/\/.*\.bestyearyet\.com\z/,  # Any subdomain
            'localhost:3000',  # Local development
            'localhost:8000'
    
    resource '/api/v1/landing_leads',
      headers: :any,
      methods: [:post, :options],
      credentials: false,
      max_age: 86400
  end
end
```

---

### 4. Rate Limiting (Recommended)

If using `rack-attack`, add to `config/initializers/rack_attack.rb`:

```ruby
# Throttle landing lead submissions by IP
Rack::Attack.throttle('landing_leads/ip', limit: 10, period: 1.minute) do |req|
  if req.path == '/api/v1/landing_leads' && req.post?
    req.ip
  end
end

# Throttle by email to prevent duplicate submissions
Rack::Attack.throttle('landing_leads/email', limit: 3, period: 1.hour) do |req|
  if req.path == '/api/v1/landing_leads' && req.post?
    # Normalize and hash the email
    email = req.params['email'].to_s.downcase.strip
    Digest::SHA256.hexdigest(email) if email.present?
  end
end
```

---

### 5. GoHighLevelService Updates (if needed)

The existing `GoHighLevelService` should already have the methods needed. If `search_contact_by_email` doesn't exist, add it:

```ruby
# Add to app/services/go_high_level_service.rb

def search_contact_by_email(email)
  response = self.class.get(
    "/contacts/",
    query: { 
      query: email, 
      locationId: @location_id 
    },
    headers: api_headers
  )
  
  handle_response(response)
  
  contacts = response.parsed_response['contacts']
  contacts&.first
end

# If create_contact doesn't accept a hash, add an overload:
def create_contact(data)
  body = {
    firstName: data[:firstName],
    lastName: data[:lastName],
    email: data[:email],
    phone: data[:phone],
    source: data[:source],
    tags: data[:tags],
    locationId: @location_id
  }.compact
  
  response = self.class.post(
    "/contacts/",
    body: body.to_json,
    headers: api_headers
  )
  
  handle_response(response)
  response.parsed_response
end

# If update_contact needs adjustment for the landing page use case:
def update_contact(contact_id, data)
  # Note: API v2 does NOT allow locationId in PUT requests
  body = {
    firstName: data[:firstName],
    lastName: data[:lastName],
    phone: data[:phone],
    tags: data[:tags]
  }.compact
  
  response = self.class.put(
    "/contacts/#{contact_id}",
    body: body.to_json,
    headers: api_headers
  )
  
  handle_response(response)
  response.parsed_response
end

private

def api_headers
  {
    'Authorization' => "Bearer #{@access_token}",
    'Version' => '2021-07-28',
    'Content-Type' => 'application/json'
  }
end
```

---

## API Specification

### Request

```http
POST /api/v1/landing_leads
Content-Type: application/json
```

**Body:**
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "phone": "+1-555-123-4567",
  "source": "signup_landing",
  "page": "new",
  "campaign": "winter_2025"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | ✅ Yes | Contact email address |
| `firstName` | string | No | First name |
| `lastName` | string | No | Last name |
| `phone` | string | No | Phone number |
| `source` | string | No | Lead source identifier (e.g., `signup_landing`, `demo_request`) |
| `page` | string | No | Landing page identifier (`new`, `gift`, `main`) |
| `campaign` | string | No | Campaign identifier for tracking |

### Response - Success (Created)

```json
{
  "success": true,
  "contact_id": "abc123xyz",
  "action": "created",
  "message": "Contact created successfully"
}
```
**Status:** `201 Created`

### Response - Success (Updated)

```json
{
  "success": true,
  "contact_id": "abc123xyz",
  "action": "updated",
  "message": "Contact updated successfully"
}
```
**Status:** `200 OK`

### Response - Validation Error

```json
{
  "success": false,
  "error": "Valid email is required"
}
```
**Status:** `422 Unprocessable Entity`

### Response - Server Error

```json
{
  "success": false,
  "error": "Failed to process lead. Please try again."
}
```
**Status:** `500 Internal Server Error`

---

## Testing

### Manual Test with cURL

```bash
# Test endpoint (replace with actual URL)
curl -X POST https://app.bestyearyet.com/api/v1/landing_leads \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Test",
    "lastName": "User",
    "email": "test-landing@example.com",
    "phone": "555-0123",
    "source": "api_test",
    "page": "new"
  }'
```

### RSpec Test

```ruby
# spec/requests/api/v1/landing_leads_spec.rb
require 'rails_helper'

RSpec.describe 'Api::V1::LandingLeads', type: :request do
  describe 'POST /api/v1/landing_leads' do
    let(:valid_params) do
      {
        firstName: 'Test',
        lastName: 'User',
        email: 'test@example.com',
        phone: '555-0123',
        source: 'test',
        page: 'new'
      }
    end

    context 'with valid parameters' do
      before do
        allow_any_instance_of(GoHighLevelService)
          .to receive(:search_contact_by_email)
          .and_return(nil)
        
        allow_any_instance_of(GoHighLevelService)
          .to receive(:create_contact)
          .and_return({ 'contact' => { 'id' => 'test123' } })
      end

      it 'creates a new contact' do
        post '/api/v1/landing_leads', params: valid_params

        expect(response).to have_http_status(:created)
        json = JSON.parse(response.body)
        expect(json['success']).to be true
        expect(json['action']).to eq 'created'
      end
    end

    context 'with missing email' do
      it 'returns validation error' do
        post '/api/v1/landing_leads', params: valid_params.except(:email)

        expect(response).to have_http_status(:unprocessable_entity)
        json = JSON.parse(response.body)
        expect(json['success']).to be false
      end
    end

    context 'with invalid email' do
      it 'returns validation error' do
        post '/api/v1/landing_leads', params: valid_params.merge(email: 'invalid')

        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
end
```

---

## Deployment Checklist

- [x] Add route to `config/routes.rb`
- [x] Create `LandingLeadsController`
- [x] Configure CORS for landing page domains
- [x] Add rate limiting rules (Redis-backed, in PublicBaseController)
- [x] Verify `GoHighLevelService` has required methods (uses existing `create_contact_raw`, `find_contact_by_exact_email`, `add_tags_to_contact`, `update_contact_raw`)
- [x] Write RSpec tests (`spec/requests/api/v1/landing_leads_spec.rb`)
- [ ] Run `bundle install` (adds `rack-cors` gem)
- [ ] Deploy to staging
- [ ] Test with cURL from staging
- [ ] Test from landing page staging environment
- [ ] Deploy to production
- [ ] Notify landing page team of endpoint URL

---

## Landing Page Integration

Once the endpoint is deployed, the landing page will call:

**Staging:** `https://byy-staging.bestyearyet.io/api/v1/landing_leads`  
**Production:** `https://bestyearyet.io/api/v1/landing_leads`

The landing page code has already been updated to use this endpoint. See `sites/new/ghl-service.js`.

---

## Security Considerations

1. **No Authentication Required** - This is intentional for public lead capture
2. **Rate Limiting** - Prevents abuse and duplicate submissions
3. **CORS Restricted** - Only allows requests from BYY domains
4. **Input Validation** - Email validation, sanitization of inputs
5. **No Sensitive Data Exposure** - Error messages don't leak internal details
6. **GHL Credentials** - Remain server-side only, never exposed to client

---

## Questions?

Contact the development team or refer to:
- `docs/GHL_PRIVATE_INTEGRATION_MIGRATION.md` - GHL API v2 details
- `docs/GOHIGHLEVEL_INTEGRATION.md` - Overall integration guide

