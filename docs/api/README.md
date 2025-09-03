# API Documentation

## Overview

The Grax Data Lake Knowledge API provides programmatic access to data management and knowledge extraction capabilities.

## Authentication

All API endpoints require authentication using API keys or OAuth 2.0.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://api.grax.com/v1/endpoint
```

## Base URL

```
https://api.grax.com/v1
```

## Core Endpoints

### Data Management

- `GET /datasets` - List all datasets
- `POST /datasets` - Create a new dataset
- `GET /datasets/{id}` - Get dataset details
- `PUT /datasets/{id}` - Update dataset
- `DELETE /datasets/{id}` - Delete dataset

### Knowledge Extraction

- `POST /knowledge/extract` - Extract knowledge from data
- `GET /knowledge/results/{job_id}` - Get extraction results
- `GET /knowledge/insights` - List available insights

### Search and Query

- `POST /search` - Search across datasets
- `POST /query` - Execute structured queries
- `GET /schema/{dataset_id}` - Get dataset schema

## Response Format

All responses are in JSON format:

```json
{
  "status": "success",
  "data": {},
  "message": "Operation completed successfully",
  "timestamp": "2024-01-01T00:00:00Z"
}
```

## Error Handling

Error responses include appropriate HTTP status codes and detailed error messages:

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": []
  }
}
```

## Rate Limiting

- Standard tier: 1000 requests/hour
- Premium tier: 10000 requests/hour
- Enterprise tier: Unlimited

Rate limit headers are included in all responses:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```
