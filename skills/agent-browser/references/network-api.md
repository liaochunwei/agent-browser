# Network Request Analysis

Obtain browser network requests, combine them with page information, and extract structured data from API responses.

**Related**: [commands.md](commands.md) for full command reference, [SKILL.md](../SKILL.md) for quick start.

## Contents

- [Basic Operation](#basic-operation)
- [Complete Workflow](#complete-workflow)
- [Use Cases](#use-cases)
- [Best Practices](#best-practices)
- [Output Format](#output-format)
- [Limitations](#limitations)

## Basic Operation

Network request monitoring is automatically enabled. Use `network requests` to view tracked requests and `network request <id>` to get full details including response bodies.

```bash
# 1. Navigate to page
agent-browser open https://api.example.com/data

# 2. Perform actions that trigger API calls
agent-browser snapshot -i
agent-browser click @e1  # Triggers API request

# 3. View tracked requests
agent-browser network requests

# 4. Get request details (includes response body)
agent-browser network request 33973.11471
```

## Complete Workflow

### Step 1: Navigate and Trigger Requests

```bash
# Open the target page
agent-browser open https://example.com/dashboard

# Wait for initial load
agent-browser wait --load networkidle

# Get page snapshot to understand the context
agent-browser snapshot -i

# Perform actions that trigger API calls (clicks, scrolls, filters, etc.)
agent-browser click @e3  # Click a button that loads data
```

### Step 2: View Tracked Requests

```bash
# List all tracked requests
agent-browser network requests

# Filter by URL pattern
agent-browser network requests --filter "api/feed"

# Filter by resource type (xhr, fetch, document, script, etc.)
agent-browser network requests --type xhr,fetch

# Filter by HTTP method
agent-browser network requests --method POST

# Filter by status code (200, 2xx, 400-499, etc.)
agent-browser network requests --status 200

# Combine filters
agent-browser network requests --filter "api" --type xhr --method GET --status 2xx
```

### Step 3: Get Request Details

```bash
# Get full request/response details by request_id
agent-browser network request 33973.11471

# Get response body directly by request_id (only response body)
agent-browser network requests --response 33973.11471 --json

# Output as JSON for programmatic processing (only requests)
agent-browser network requests --filter "api/data" --json > requests.json
```

### Step 4: Extract Structured Data

When you have the response body, you can parse and extract the data you need:

```bash
# Get JSON response and extract specific fields with jq
agent-browser network requests --filter "api/users" --json | jq '.data.requests[].url'

# Get all product data from an API response
agent-browser network requests --filter "api/products" --json | jq '.data.requests[] | select(.resourceType == "xhr") | {url, status, timestamp}'
```

## Use Cases

### Extract API Response Data

This is the most common workflow: identify the API endpoint, get its response, and extract the data you need.

```bash
#!/bin/bash
# Extract user data from API response

# 1. Navigate to page
agent-browser open https://example.com/admin/users

# 2. Wait for data to load
agent-browser wait --load networkidle

# 3. Find the API request that fetches users
# Filter for XHR requests with "users" in the URL
agent-browser network requests --filter "users" --type xhr --json > /tmp/requests.json

# 4. Find the requestId for the users API
REQUEST_ID=$(cat /tmp/requests.json | jq '.data.requests[] | select(.url | contains("api/users")) | .requestId')

# 5. Get the response body
agent-browser network requests --response "$REQUEST_ID" --json > /tmp/user_response.json

# 6. Extract the data you need
cat /tmp/user_response.json | jq '.data.body' | jq '.users[].name'
```

### Scrape Dynamic Content (AJAX/SPA)

Many modern web apps load content via API calls. Use network monitoring to capture this data.

```bash
#!/bin/bash
# Scrape product listings from a dynamic site

agent-browser open https://shop.example.com/category/electronics
agent-browser wait --load networkidle

# Click to load more products (pagination)
agent-browser snapshot -i
agent-browser click @e5  # "Load More" button
agent-browser wait --load networkidle

# Capture all product API requests
agent-browser network requests --filter "/api/products" --type xhr --json > /tmp/products.json

# Extract product data from responses
for req_id in $(cat /tmp/products.json | jq -r '.data.requests[].requestId'); do
    agent-browser network requests --response "$req_id" --json | jq '.data.body.products[]'
done
```

### Analyze Chart/Graph Data

Canvas charts often fetch data from APIs. Capture and reconstruct the data.

```bash
#!/bin/bash
# Extract data used to render a chart

agent-browser open https://example.com/analytics

# Wait for chart to render (it fetches data first)
agent-browser wait --load networkidle

# Find requests that return chart data
agent-browser network requests --filter "analytics" --type xhr --json

# Get specific request details
# Use the request_id from the output above
agent-browser network request 12345.67890 --json
```

### Monitor Form Submissions

Track what data your forms actually send to the server.

```bash
#!/bin/bash
# Monitor form submission payload

agent-browser open https://example.com/submit

# Fill the form
agent-browser fill @e1 "John Doe"
agent-browser fill @e2 "john@example.com"
agent-browser fill @e3 "This is my message"

# Find the POST request
agent-browser network requests --filter "submit" --method POST --json

# Get the request details including POST body
# Replace REQUEST_ID with actual value
agent-browser network request REQUEST_ID --json | jq '.postData'
```

## Best Practices

### 1. Filter Early, Get Details Later

Always filter requests first to identify the correct `request_id`, then fetch details.

```bash
# WRONG: Get all requests, then search through them
agent-browser network requests --json | jq '.data.requests[]'

# RIGHT: Filter to find the specific request first
agent-browser network requests --filter "api/feed" --type xhr --json
# Then use the request_id from this output
agent-browser network requests --response REQUEST_ID --json
```

### 2. Use JSON Output for Scripting

Always use `--json` flag when processing requests programmatically.

```bash
# Parse with jq for extraction
agent-browser network requests --filter "api" --json | jq '.data.requests[].url'

# Save for later analysis
agent-browser network requests --filter "api" --type xhr --json > api_requests.json
```

### 3. Combine with Page Context

Use snapshot alongside network requests to understand the page structure.

```bash
# Get page snapshot to understand what triggered the request
agent-browser snapshot -i
# Note which element @eN triggered the API call

# Then check what requests were made
agent-browser network requests --filter "api" --json

# This helps correlate UI actions with network activity
```

### 4. Clear Cache When Needed

Start fresh if you're getting cached responses.

```bash
# Clear tracked requests
agent-browser network requests --clear

# Reload the page
agent-browser reload
agent-browser wait --load networkidle

# Now you see only the new requests
agent-browser network requests --filter "api" --json
```

### 5. Use HAR Recording for Complex Sessions

For complex multi-step workflows, use HAR recording.

```bash
# Start recording
agent-browser network har start

# Perform your workflow
agent-browser click @e1
agent-browser wait --load networkidle
agent-browser click @e2
agent-browser fill @e3 "search query"

# Stop and save
agent-browser network har stop ./session.har

# Analyze the HAR file with tools like Chrome DevTools or har-viewer
```

## Output Format

### Requests List Response

```json
{
  "success": true,
  "data": {
    "requests": [
      {
        "requestId": "33973.11471",
        "url": "https://example.com/api/feed",
        "method": "GET",
        "headers": {
          "Accept": "application/json",
          "Authorization": "Bearer token123"
        },
        "resourceType": "xhr",
        "response": {
          "status": 200,
          "contentType": "application/json; charset=utf-8",
          "contentSize": 1024
        },
        "timestamp": 1773641916654
      }
    ]
  },
  "error": null
}
```

### Response Body Response

```json
{
  "success": true,
  "data": {
    "body": "{\"users\":[{\"id\":1,\"name\":\"Alice\"},{\"id\":2,\"name\":\"Bob\"}]}"
  },
  "error": null
}
```

### Request Detail Response

Includes both request metadata and response body in a single response.

```json
{
  "success": true,
  "data": {
    "requestId": "33973.11471",
    "url": "https://example.com/api/users",
    "method": "GET",
    "headers": {
      "Accept": "application/json"
    },
    "resourceType": "xhr",
    "response": {
      "status": 200,
      "contentType": "application/json",
      "contentSize": 512
    },
    "timestamp": 1773641916654,
    "responseBody": "{\"users\":[{\"id\":1,\"name\":\"Alice\",\"email\":\"alice@example.com\"},{\"id\":2,\"name\":\"Bob\",\"email\":\"bob@example.com\"}],\"total\":2}"
  },
  "error": null
}
```

**Key fields:**
- `requestId` — The unique identifier for matching request to response (use this for `network request <id>`)
- `resourceType` — Request type: `xhr`, `fetch`, `document`, `script`, `stylesheet`, `image`, etc.
- `response.contentType` — MIME type of response (e.g., `application/json`)
- `responseBody` — Raw response body string (JSON string when contentType is `application/json`)

## Limitations

- **Binary responses**: Images, PDFs, and other binary content may be truncated or show `[base64, N chars]` instead of actual content.

- **Large recordings**: HAR files can grow large with extended sessions. Use `--clear` periodically to reset tracked requests.

- **Timing**: Requests are tracked as they occur. If a request completes before you start monitoring, it won't appear in the list.

- **CORS restrictions**: Requests blocked by CORS policy won't appear in tracked requests.

- **Streaming responses**: Server-sent events (SSE) and chunked transfer encoding may not capture complete data.

- **WebSocket**: WebSocket connections are not tracked in the request list.
