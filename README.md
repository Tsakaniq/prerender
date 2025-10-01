# CSN Prerender Service

A production-ready Node.js service that prerenders React SPA pages for better SEO by fetching and serving fully rendered HTML content with advanced caching and multi-domain support.

## Problem Statement

Create React App builds single-page applications (SPAs) where SEO meta tags are not dynamic until the page is fully loaded. This service solves this issue by prerendering pages and serving them with proper meta tags for search engines and social media crawlers.

## How It Works

1. **Request Interception**: The service receives requests for specific paths (configured via ALB)
2. **Multi-Domain Detection**: Automatically detects target domain from ALB Host header
3. **Page Prerendering**: Uses Puppeteer to fetch the full HTML from the React SPA
4. **Advanced Caching**: Caches prerendered content for 30 days with domain separation
5. **Response**: Returns fully rendered HTML with proper meta tags

## Architecture

```
User Request → ALB → Prerender Service → React SPA → Prerendered HTML
                     ↓
                 Cache Layer (30-day TTL)
```

## Project Structure

```
src/
├── config/
│   └── index.js          # Configuration management
├── middleware/
│   └── index.js          # Express middleware setup
├── services/
│   ├── cache.js          # Cache service with domain separation
│   └── browser.js        # Puppeteer browser management
├── routes/
│   ├── health.js         # Health check endpoints
│   ├── prerender.js      # Main prerender logic
│   └── index.js          # Route aggregation
├── utils/
│   ├── logger.js         # Professional logging system
│   └── gracefulShutdown.js # Graceful shutdown handling
├── app.js                # Express app factory
└── server.js             # Application entry point
```

## Installation

1. Install dependencies:
```bash
npm install
```

2. Configure environment variables (copy from `config.example.env`):
```bash
cp config.example.env .env
```

3. Start the service:
```bash
# Development
npm run dev

# Production
npm start
```

## Configuration

### Required Environment Variables

**None!** The service uses Host header from ALB for domain detection.

### Optional Environment Variables

- `PORT`: Server port (default: 3000)
- `NODE_ENV`: Environment (development/production, default: development)

### Domain Detection Logic

The service uses **automatic domain detection** from the ALB Host header:

1. **Primary Method**: Uses `Host` header from ALB (automatic domain detection)
   - When ALB routes `yourdomain.com/blog/post` → service prerenders `https://yourdomain.com/blog/post`
   - When ALB routes `app.yourdomain.com/courses/101` → service prerenders `https://app.yourdomain.com/courses/101`

2. **No Fallback**: Host header is mandatory for multi-domain safety

### ALB Configuration Example

```yaml
# ALB Rules - Host header automatically sent to service
- Host: yourdomain.com
  Path: /blog/*
  Target: Prerender Service
  # Service receives: Host: yourdomain.com

- Host: app.yourdomain.com
  Path: /courses/*
  Target: Prerender Service
  # Service receives: Host: app.yourdomain.com
```

### Cache Configuration

- **TTL**: 30 days (2,592,000 seconds) - optimized for static content
- **Check Period**: 24 hours - automatic cleanup of expired entries
- **Max Keys**: 1000 - prevents memory issues
- **Domain Separation**: Each domain has separate cache namespace

### Browser Configuration

- **Headless Mode**: New Chromium headless mode
- **Viewport**: 1920x1080 for consistent rendering
- **Timeout**: 30 seconds for page loading
- **User Agent**: Chrome 120 for compatibility

## API Endpoints

### `GET /health`
Health check endpoint for ALB monitoring.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "mode": "multi-domain",
  "hostHeaderRequired": true
}
```

### `GET /*`
Prerenders any path and returns fully rendered HTML.

**Headers:**
- `X-Cache`: HIT/MISS (indicates if content was served from cache)
- `X-Cache-Key`: Cache key used (includes domain)
- `X-Cache-TTL`: Cache time-to-live (2592000 seconds)
- `X-Cache-Expires`: When cache expires (ISO timestamp)
- `Cache-Control`: Browser caching instructions (30 days)
- `X-Prerender`: true (indicates response from prerender service)
- `X-Prerender-Timestamp`: When content was prerendered
- `X-Target-Domain`: Which domain was used
- `X-Response-Time`: Request processing time

## Features

- ✅ **Multi-Domain Support**: Automatic domain detection from ALB
- ✅ **Advanced Caching**: 30-day cache with domain separation
- ✅ **Professional Logging**: Structured JSON logs with real-time troubleshooting
- ✅ **Puppeteer Integration**: Headless Chrome with optimized configuration
- ✅ **Error Handling**: Graceful fallbacks and comprehensive error logging
- ✅ **Performance**: Compression, security headers, and browser reuse
- ✅ **Production Ready**: Modular architecture with proper separation of concerns
- ✅ **Graceful Shutdown**: Proper cleanup on termination signals
- ✅ **Multi-Domain Safe**: No fallback risks, Host header mandatory

## Logging System

### Production Logging
Structured JSON logs with essential operational data:

```json
[2024-01-01T12:00:00.000Z] INFO: Request RECEIVED | {"method":"GET","path":"/blog/post","host":"yourdomain.com","hasHostHeader":true}

[2024-01-01T12:00:01.000Z] INFO: Cache MISS | {"operation":"MISS","key":"prerender:yourdomain.com:/blog/post","hostHeader":"yourdomain.com","requestPath":"/blog/post"}

[2024-01-01T12:00:02.500Z] INFO: Browser PRERENDERING_COMPLETE | {"operation":"PRERENDERING_COMPLETE","url":"https://yourdomain.com/blog/post","hostHeader":"yourdomain.com","responseTime":"1500ms","contentSize":"45678 bytes"}
```

### Development Logging
Includes DEBUG logs for detailed troubleshooting:

```json
[2024-01-01T12:00:00.000Z] DEBUG: Starting page prerendering | {"url":"https://yourdomain.com","viewport":{"width":1920,"height":1080},"timeout":30000}
```

### Log Levels
- **INFO**: Request processing, cache operations, browser lifecycle
- **WARN**: Non-critical issues (missing selectors, etc.)
- **ERROR**: Critical errors with full context and stack traces
- **DEBUG**: Detailed operations (development only)

## Performance Considerations

- **Caching**: Prerendered content cached for 30 days (optimized for static content)
- **Browser Reuse**: Single browser instance reused across requests
- **Compression**: Responses are gzipped for faster transfer
- **Timeout Handling**: Configurable timeouts prevent hanging requests
- **Memory Management**: Automatic cleanup of expired cache entries
- **Domain Separation**: Prevents cache conflicts between domains

## Deployment

### Docker (Recommended)

```bash
# Build the image
docker build -t csn-prerender .

# Run the service
docker run -p 3000:3000 csn-prerender

# Or with docker-compose
docker-compose up
```

### Docker Compose

```yaml
version: '3.8'
services:
  prerender:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### PM2

```bash
npm install -g pm2
pm2 start src/server.js --name "prerender-service" --env production
```

## Monitoring

### Health Checks
- **Endpoint**: `GET /health`
- **Response**: Service status and configuration
- **ALB Integration**: Ready for load balancer health checks

### Log Monitoring
- **Structured Logs**: JSON format for log aggregation
- **Real-time Troubleshooting**: Host header detection, cache performance
- **Performance Metrics**: Response times, cache hit rates
- **Error Tracking**: Comprehensive error logging with context

### Cache Monitoring
- **Hit/Miss Rates**: Track cache performance
- **Content Sizes**: Monitor memory usage
- **Domain Separation**: Verify multi-domain functionality

## Troubleshooting

### Common Issues

1. **Missing Host Header**: Service returns 400 Bad Request with clear error message
2. **Browser Launch Failures**: Check Docker permissions and Chromium installation
3. **Timeout Errors**: Monitor response times in logs
4. **Cache Issues**: Restart service to clear cache (30-day TTL)

### Log Analysis

**Cache Performance:**
```bash
# View cache hit rates
docker logs csn-prerender | grep "Cache HIT"

# Monitor cache misses
docker logs csn-prerender | grep "Cache MISS"
```

**Domain Detection:**
```bash
# Verify host header detection
docker logs csn-prerender | grep "hasHostHeader"

# Check target domain resolution
docker logs csn-prerender | grep "targetDomain"
```

**Performance Monitoring:**
```bash
# Monitor response times
docker logs csn-prerender | grep "responseTime"

# Track content sizes
docker logs csn-prerender | grep "contentSize"
```

### Error Investigation

**Configuration Errors:**
```bash
# Check for missing host headers
docker logs csn-prerender | grep "Missing Host header"
```

**Browser Errors:**
```bash
# Monitor browser initialization
docker logs csn-prerender | grep "Browser"

# Check prerendering failures
docker logs csn-prerender | grep "PRERENDERING_FAILED"
```

## Security

- **Helmet.js**: Security headers for production
- **CORS**: Cross-origin request handling
- **Non-root User**: Docker container runs as non-root user
- **Input Validation**: Proper request validation and sanitization
- **Error Handling**: No sensitive data in error responses
- **Logging**: No sensitive data in logs

## Development

### Local Development

```bash
# Install dependencies
npm install

# Start in development mode
npm run dev
```

### Code Structure

- **Modular Architecture**: Clean separation of concerns
- **Service Layer**: Business logic separated from routes
- **Configuration**: Centralized environment management
- **Logging**: Professional logging with structured output
- **Error Handling**: Comprehensive error management

## License

ISC

## Support

For issues and questions, please check the troubleshooting section or create an issue in the repository.