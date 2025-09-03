# Installation Guide

## Prerequisites

- Node.js 18+ or Python 3.9+
- Docker (optional, for containerized deployment)
- Access to cloud storage (AWS S3, Azure Blob, or GCP Cloud Storage)
- Database (PostgreSQL 13+ or MongoDB 5+)

## Installation Methods

### Method 1: Docker Compose (Recommended)

1. Clone the repository:
   ```bash
   git clone https://github.com/graxinc/grax-datalake-knowledge.git
   cd grax-datalake-knowledge
   ```

2. Copy environment configuration:
   ```bash
   cp .env.example .env
   ```

3. Update environment variables in `.env`:
   ```env
   DATABASE_URL=postgresql://user:password@localhost:5432/grax_db
   STORAGE_PROVIDER=s3
   AWS_ACCESS_KEY_ID=your_access_key
   AWS_SECRET_ACCESS_KEY=your_secret_key
   S3_BUCKET_NAME=your-bucket-name
   ```

4. Start services:
   ```bash
   docker-compose up -d
   ```

### Method 2: Manual Installation

1. Install dependencies:
   ```bash
   npm install
   # or
   pip install -r requirements.txt
   ```

2. Initialize database:
   ```bash
   npm run db:migrate
   # or
   python manage.py migrate
   ```

3. Start the application:
   ```bash
   npm start
   # or
   python app.py
   ```

## Configuration

### Environment Variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `PORT` | Application port | No | 3000 |
| `DATABASE_URL` | Database connection string | Yes | - |
| `STORAGE_PROVIDER` | Storage provider (s3/azure/gcp) | Yes | - |
| `LOG_LEVEL` | Logging level | No | info |
| `API_KEY_SECRET` | Secret for API key generation | Yes | - |

### Storage Configuration

#### AWS S3
```env
STORAGE_PROVIDER=s3
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
S3_BUCKET_NAME=your-bucket-name
AWS_REGION=us-east-1
```

#### Azure Blob Storage
```env
STORAGE_PROVIDER=azure
AZURE_STORAGE_ACCOUNT=your_account
AZURE_STORAGE_KEY=your_key
AZURE_CONTAINER_NAME=your-container
```

## Verification

After installation, verify the system is working:

1. Check health endpoint:
   ```bash
   curl http://localhost:3000/health
   ```

2. Test API access:
   ```bash
   curl -H "Authorization: Bearer YOUR_API_KEY" \
     http://localhost:3000/api/v1/datasets
   ```

## Next Steps

- Review the [User Guide](../user-guides/getting-started.md)
- Check the [API Documentation](../api/README.md)
- Set up monitoring and logging
