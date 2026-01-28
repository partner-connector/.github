# HubSpot Webhook Processing Guide

**Last Updated:** January 28, 2026
**Version:** 1.0
**Status:** Production

---

## Table of Contents

1. [Overview](#overview)
2. [Webhook Configuration](#webhook-configuration)
3. [Event Processing Architecture](#event-processing-architecture)
4. [Contact Property Change Flow](#contact-property-change-flow)
5. [Deal Property Change Flow](#deal-property-change-flow)
6. [Signature Validation](#signature-validation)
7. [Debugging Webhooks](#debugging-webhooks)
8. [Common Issues](#common-issues)

---

## Overview

> **Important**: This guide documents **Integration 2 (Marketplace OAuth)** webhooks only.
>
> This covers webhooks from **partner's HubSpot CRM** to Partner Connector (deal/contact changes).
>
> For ChiliPiper webhook routing (Integration 1), see `api/src/modules/meetings/` - this is a separate flow.

### What Are Webhooks?

Webhooks are **real-time notifications** from HubSpot when data changes in a partner's CRM. Instead of polling HubSpot every few minutes, webhooks push changes instantly.

**Benefits:**
- ✅ **Real-time sync** - Changes appear immediately
- ✅ **Efficient** - No unnecessary API calls
- ✅ **Scalable** - HubSpot handles delivery

---

### Event Types

| Event Type | Trigger | Partner Connector Action |
|------------|---------|--------------------------|
| `contact.propertyChange` | Contact field updated | Update lead record |
| `deal.propertyChange` | Deal field updated | Update lead status/stage |
| `contact.deletion` | Contact deleted (future) | Mark lead as deleted |
| `deal.deletion` | Deal deleted (future) | Remove marketplace listing |

**Currently Implemented:** `contact.propertyChange`, `deal.propertyChange`

---

### Webhook Flow Overview

```mermaid
sequenceDiagram
    participant HubSpot as HubSpot CRM
    participant Guard as Webhook Guard
    participant Controller as API Controller
    participant Queue as BullMQ Queue
    participant Processor as Webhook Processor
    participant DB as MongoDB

    HubSpot->>Guard: POST /v1/leads/webhooks/hubspot
    Note over HubSpot,Guard: Headers: x-hubspot-signature
    Guard->>Guard: Validate signature
    alt Signature Invalid
        Guard-->>HubSpot: 401 Unauthorized
    else Signature Valid
        Guard->>Controller: Forward request
        Controller->>Queue: Add to hubspot-webhook queue
        Controller-->>HubSpot: 200 OK (non-blocking)
        Queue->>Processor: Process event asynchronously
        Processor->>DB: Update lead record
        Processor->>DB: Log sync event
    end
```

---

## Webhook Configuration

### Location

**Repository:** `partner-connector-hubspot-app`
**File:** `src/app/app.json`

---

### Configuration Example

```json
{
  "name": "Partner Connector",
  "webhooks": {
    "targetUrl": "https://api.partners.belkins.io/v1/leads/webhooks/hubspot",
    "subscriptions": [
      {
        "subscriptionType": "contact.propertyChange",
        "propertyName": "*"
      },
      {
        "subscriptionType": "deal.propertyChange",
        "propertyName": "*"
      }
    ]
  }
}
```

---

### Subscription Breakdown

**1. Contact Property Change**
```json
{
  "subscriptionType": "contact.propertyChange",
  "propertyName": "*"
}
```

**Triggers on:**
- Email updated
- Phone number changed
- Company updated
- Custom property modified
- **Any** contact field change (`*` = all properties)

---

**2. Deal Property Change**
```json
{
  "subscriptionType": "deal.propertyChange",
  "propertyName": "*"
}
```

**Triggers on:**
- Deal stage changed
- Deal amount updated
- Close date modified
- Custom property changed
- **Any** deal field change (`*` = all properties)

---

### Environment-Specific URLs

| Environment | Webhook Target URL |
|-------------|--------------------|
| **Local** | `http://localhost:3000/v1/leads/webhooks/hubspot` |
| **Staging** | `https://staging-api.partners.belkins.io/v1/leads/webhooks/hubspot` |
| **Production** | `https://api.partners.belkins.io/v1/leads/webhooks/hubspot` |

**Note:** HubSpot requires HTTPS for webhooks (HTTP only allowed for localhost testing)

---

### Setting Webhook Secret

```bash
# 1. Generate random secret
openssl rand -hex 32
# Output: a1b2c3d4e5f6...

# 2. Add to HubSpot app settings
# Go to: https://app.hubspot.com/developer/{accountId}/applications
# Navigate to: Webhooks → Webhook Secret
# Paste generated secret

# 3. Add to API .env
echo "HUBSPOT_WEBHOOK_SECRET=a1b2c3d4e5f6..." >> api/.env

# 4. Update Kubernetes secret (production)
kubectl create secret generic partner-connector-secrets \
  --from-literal=hubspot-webhook-secret='a1b2c3d4e5f6...' \
  --namespace partner-connector \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## Event Processing Architecture

### Components

```
HubSpot CRM
    ↓
[1] Webhook Guard - Signature validation
    ↓
[2] Controller - Accept webhook (200 OK immediately)
    ↓
[3] BullMQ Queue - Asynchronous processing
    ↓
[4] Processor - Business logic
    ↓
[5] MongoDB - Persist changes
```

---

### 1. Webhook Guard (Signature Validation)

**File:** `src/common/guards/hubspot-webhook.guard.ts`

```typescript
@Injectable()
export class HubSpotWebhookGuard implements CanActivate {
  private readonly logger = new Logger(HubSpotWebhookGuard.name);

  constructor(private readonly configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context
      .switchToHttp()
      .getRequest<RawBodyRequest<Request>>();

    const signature = request.headers['x-hubspot-signature'] as string;
    const secret = this.configService.get<string>('HUBSPOT_WEBHOOK_SECRET');

    if (!signature) {
      this.logger.error('Missing x-hubspot-signature header');
      throw new UnauthorizedException('Missing webhook signature');
    }

    if (!secret) {
      this.logger.error('HUBSPOT_WEBHOOK_SECRET not configured');
      throw new InternalServerErrorException('Webhook secret not configured');
    }

    // Extract signature version and hash
    // Format: v3=sha256hash...
    const [version, hash] = signature.split('=');
    if (version !== 'v3') {
      this.logger.error(`Unsupported signature version: ${version}`);
      throw new UnauthorizedException('Unsupported signature version');
    }

    // Get raw body (CRITICAL: must use raw body, not JSON.stringify)
    if (!request.rawBody) {
      this.logger.error('rawBody not available - ensure rawBody: true in main.ts');
      throw new UnauthorizedException('Raw body not available');
    }

    const body = request.rawBody.toString('utf8');
    const timestamp = request.headers['x-hubspot-request-timestamp'] as string;

    // Construct source string for signature
    const sourceString = `v3:${secret}:${timestamp}:${body}`;

    // Calculate expected signature
    const expectedHash = crypto
      .createHash('sha256')
      .update(sourceString)
      .digest('hex');

    // Compare signatures (constant-time comparison)
    const isValid = crypto.timingSafeEqual(
      Buffer.from(hash),
      Buffer.from(expectedHash),
    );

    if (!isValid) {
      this.logger.error('Invalid webhook signature');
      throw new UnauthorizedException('Invalid webhook signature');
    }

    // Signature valid
    return true;
  }
}
```

**Key Points:**
- ✅ Uses `rawBody` (not `JSON.stringify(body)`)
- ✅ Constant-time comparison prevents timing attacks
- ✅ Validates signature version (v3)
- ✅ Includes timestamp in signature calculation

---

### 2. Controller (Webhook Endpoint)

**File:** `src/modules/leads/leads.controller.ts`

```typescript
@Controller({ path: 'leads', version: '1' })
export class LeadsController {
  constructor(
    @InjectQueue(QUEUES.HUBSPOT_WEBHOOK) private readonly webhookQueue: Queue,
  ) {}

  @Post('webhooks/hubspot')
  @UseGuards(HubSpotWebhookGuard) // Signature validation
  @HttpCode(HttpStatus.OK)
  async handleHubSpotWebhook(@Body() payload: any) {
    // Add to queue for async processing
    await this.webhookQueue.add('hubspot-event', payload, {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 5000, // Start with 5 seconds
      },
    });

    // Return 200 OK immediately (non-blocking)
    return { received: true };
  }
}
```

**Key Points:**
- ✅ Returns 200 OK immediately (HubSpot timeout = 5 seconds)
- ✅ Async processing via queue (prevents timeout)
- ✅ 3 retry attempts with exponential backoff
- ✅ Signature validation via guard

---

### 3. BullMQ Queue

**Queue Name:** `hubspot-webhook`
**Purpose:** Decouple webhook receipt from processing

**Configuration:**
```typescript
// In LeadsModule
BullModule.registerQueue({
  name: QUEUES.HUBSPOT_WEBHOOK,
  defaultJobOptions: {
    removeOnComplete: 100, // Keep last 100 completed jobs
    removeOnFail: 500,     // Keep last 500 failed jobs
  },
})
```

---

### 4. Webhook Processor

**File:** `src/modules/leads/hubspot-webhook.processor.ts`

```typescript
@Processor(QUEUES.HUBSPOT_WEBHOOK)
export class HubSpotWebhookProcessor extends WorkerHost {
  private readonly logger = new Logger(HubSpotWebhookProcessor.name);

  constructor(
    private readonly leadsService: LeadsService,
    private readonly syncRepository: SyncRepository,
  ) {
    super();
  }

  async process(job: Job<HubSpotWebhookPayload>) {
    const { data } = job;
    const { subscriptionType, objectId, propertyName, propertyValue } = data;

    this.logger.log(`Processing ${subscriptionType} event for object ${objectId}`);

    try {
      if (subscriptionType === 'contact.propertyChange') {
        await this.handleContactPropertyChange(data);
      } else if (subscriptionType === 'deal.propertyChange') {
        await this.handleDealPropertyChange(data);
      } else {
        this.logger.warn(`Unknown subscription type: ${subscriptionType}`);
      }

      // Log successful sync
      await this.syncRepository.create({
        partnerId: data.portalId,
        taskType: 'webhook',
        status: 'success',
        recordsProcessed: 1,
      });
    } catch (error) {
      this.logger.error(`Webhook processing failed: ${error.message}`, error.stack);

      // Log failed sync
      await this.syncRepository.create({
        partnerId: data.portalId,
        taskType: 'webhook',
        status: 'error',
        error: error.message,
      });

      throw error; // Trigger BullMQ retry
    }
  }

  private async handleContactPropertyChange(data: HubSpotWebhookPayload) {
    const { objectId, propertyName, propertyValue } = data;

    // Find lead by HubSpot contact ID
    const lead = await this.leadsService.findByHubSpotId(objectId);
    if (!lead) {
      this.logger.warn(`Lead not found for HubSpot contact ${objectId}`);
      return;
    }

    // Map HubSpot property to lead field
    const updateFields: any = {};

    switch (propertyName) {
      case 'email':
        updateFields.email = propertyValue;
        break;
      case 'firstname':
        updateFields.firstName = propertyValue;
        break;
      case 'lastname':
        updateFields.lastName = propertyValue;
        break;
      case 'phone':
        updateFields.phone = propertyValue;
        break;
      case 'company':
        updateFields.company = propertyValue;
        break;
      // ... more mappings
    }

    if (Object.keys(updateFields).length > 0) {
      await this.leadsService.update(lead._id, updateFields);
      this.logger.log(`Updated lead ${lead._id} with contact changes`);
    }
  }

  private async handleDealPropertyChange(data: HubSpotWebhookPayload) {
    const { objectId, propertyName, propertyValue } = data;

    // Find lead by HubSpot deal ID
    const lead = await this.leadsService.findByHubSpotDealId(objectId);
    if (!lead) {
      this.logger.warn(`Lead not found for HubSpot deal ${objectId}`);
      return;
    }

    // Map HubSpot deal stage to lead status
    if (propertyName === 'dealstage') {
      const status = this.mapDealStageToStatus(propertyValue);
      await this.leadsService.update(lead._id, { status });
      this.logger.log(`Updated lead ${lead._id} status to ${status}`);
    }
  }

  private mapDealStageToStatus(dealStage: string): string {
    // Map HubSpot deal stages to PC statuses
    const stageMap = {
      'appointmentscheduled': 'scheduled',
      'qualifiedtobuy': 'in_touch',
      'closedwon': 'won',
      'closedlost': 'lost_no_potential',
      // ... more mappings
    };
    return stageMap[dealStage.toLowerCase()] || 'new';
  }
}
```

---

## Contact Property Change Flow

### Detailed Flow

```mermaid
sequenceDiagram
    participant HubSpot as HubSpot CRM
    participant API as API (Webhook Endpoint)
    participant Queue as BullMQ Queue
    participant Processor as Webhook Processor
    participant LeadsService as Leads Service
    participant DB as MongoDB

    Note over HubSpot: Partner updates contact email
    HubSpot->>API: POST /v1/leads/webhooks/hubspot
    Note over HubSpot,API: Payload:<br/>{<br/>  subscriptionType: "contact.propertyChange",<br/>  objectId: 12345,<br/>  propertyName: "email",<br/>  propertyValue: "new@example.com"<br/>}

    API->>API: Validate signature
    API->>Queue: Add job to queue
    API-->>HubSpot: 200 OK

    Queue->>Processor: Trigger process()
    Processor->>LeadsService: findByHubSpotId(12345)
    LeadsService->>DB: Find lead
    DB-->>LeadsService: Lead document
    LeadsService-->>Processor: Lead found

    Processor->>Processor: Map property (email → email)
    Processor->>LeadsService: update(leadId, { email: "new@example.com" })
    LeadsService->>DB: Update lead.email
    DB-->>LeadsService: Updated

    Processor->>DB: Log sync success
```

---

### Webhook Payload Example

```json
{
  "subscriptionType": "contact.propertyChange",
  "eventId": 1234567890,
  "subscriptionId": 123,
  "portalId": 987654,
  "occurredAt": 1706482800000,
  "objectId": 12345,
  "propertyName": "email",
  "propertyValue": "john.doe@example.com",
  "changeSource": "CRM",
  "sourceId": "userId:123"
}
```

---

### Property Mapping

| HubSpot Property | PC Lead Field | Notes |
|------------------|---------------|-------|
| `email` | `email` | Primary identifier |
| `firstname` | `firstName` | |
| `lastname` | `lastName` | |
| `phone` | `phone` | |
| `company` | `company` | |
| `jobtitle` | `jobTitle` | |
| `country_all` | `country` | Custom property |
| `industry_website` | `industry` | Custom property |
| `company_size_new` | `companySize` | Custom property |
| `meeting_type_cp__c` | `meetingType` | ChiliPiper meeting type |

---

## Deal Property Change Flow

### Detailed Flow

```mermaid
sequenceDiagram
    participant HubSpot as HubSpot CRM
    participant API as API (Webhook Endpoint)
    participant Queue as BullMQ Queue
    participant Processor as Webhook Processor
    participant LeadsService as Leads Service
    participant DB as MongoDB

    Note over HubSpot: Partner moves deal to "Closed Won"
    HubSpot->>API: POST /v1/leads/webhooks/hubspot
    Note over HubSpot,API: Payload:<br/>{<br/>  subscriptionType: "deal.propertyChange",<br/>  objectId: 67890,<br/>  propertyName: "dealstage",<br/>  propertyValue: "closedwon"<br/>}

    API->>API: Validate signature
    API->>Queue: Add job to queue
    API-->>HubSpot: 200 OK

    Queue->>Processor: Trigger process()
    Processor->>LeadsService: findByHubSpotDealId(67890)
    LeadsService->>DB: Find lead
    DB-->>LeadsService: Lead document
    LeadsService-->>Processor: Lead found

    Processor->>Processor: Map deal stage (closedwon → won)
    Processor->>LeadsService: update(leadId, { status: "won" })
    LeadsService->>DB: Update lead.status
    DB-->>LeadsService: Updated

    Processor->>DB: Log sync success
```

---

### Webhook Payload Example

```json
{
  "subscriptionType": "deal.propertyChange",
  "eventId": 1234567891,
  "subscriptionId": 124,
  "portalId": 987654,
  "occurredAt": 1706482900000,
  "objectId": 67890,
  "propertyName": "dealstage",
  "propertyValue": "closedwon",
  "changeSource": "CRM",
  "sourceId": "userId:123"
}
```

---

### Deal Stage Mapping

| HubSpot Deal Stage | PC Lead Status | Marketplace Action |
|--------------------|----------------|--------------------|
| `appointmentscheduled` | `scheduled` | Keep listed |
| `qualifiedtobuy` | `in_touch` | Keep listed |
| `presentationscheduled` | `in_touch` | Keep listed |
| `closedwon` | `won` | **Remove from marketplace** |
| `closedlost` | `lost_no_potential` | **Re-list on marketplace** |

**Business Rule:** Won leads are removed from marketplace (partner closed the deal)

---

## Signature Validation

### Why Validate Signatures?

Without signature validation, attackers could:
- ❌ Send fake webhooks to corrupt data
- ❌ Spam your endpoint with requests
- ❌ Trigger expensive operations

With signature validation:
- ✅ Only authentic HubSpot webhooks accepted
- ✅ Tampering detected immediately
- ✅ Replay attacks prevented (timestamp check)

---

### Signature Format

```
x-hubspot-signature: v3=a1b2c3d4e5f6...
```

**Components:**
- `v3` = Signature version (HubSpot standard)
- `a1b2c3d4e5f6...` = SHA-256 hash

---

### Signature Calculation

```typescript
// HubSpot calculates signature like this:
const sourceString = `v3:${secret}:${timestamp}:${rawBody}`;
const signature = crypto.createHash('sha256').update(sourceString).digest('hex');

// We validate by recalculating and comparing:
const expectedSignature = crypto.createHash('sha256').update(sourceString).digest('hex');
const isValid = crypto.timingSafeEqual(
  Buffer.from(receivedSignature),
  Buffer.from(expectedSignature),
);
```

---

### Critical: Use Raw Body

❌ **WRONG - Will always fail:**
```typescript
const body = JSON.stringify(request.body); // ❌ Different formatting
const sourceString = `v3:${secret}:${timestamp}:${body}`;
```

✅ **CORRECT:**
```typescript
const body = request.rawBody.toString('utf8'); // ✅ Exact bytes
const sourceString = `v3:${secret}:${timestamp}:${body}`;
```

**Why?** `JSON.stringify()` may format differently than original:
- Whitespace differences: `{"key": "value"}` vs `{"key":"value"}`
- Property order may change
- Number formatting: `1.0` vs `1`

---

### Timestamp Validation (Future Enhancement)

```typescript
// Reject old webhooks (prevent replay attacks)
const timestamp = parseInt(request.headers['x-hubspot-request-timestamp'] as string);
const now = Date.now();
const age = now - timestamp;

if (age > 5 * 60 * 1000) { // 5 minutes
  throw new UnauthorizedException('Webhook too old');
}
```

---

## Debugging Webhooks

### 1. Enable Detailed Logging

```typescript
// In hubspot-webhook.processor.ts
async process(job: Job<HubSpotWebhookPayload>) {
  const { data } = job;

  // Log full payload
  this.logger.debug('Webhook payload:', JSON.stringify(data, null, 2));

  // Log attempt number
  this.logger.log(`Processing attempt ${job.attemptsMade + 1} of ${job.opts.attempts}`);

  // ... process webhook
}
```

---

### 2. Check BullMQ Dashboard

```bash
# Install Bull Board (development only)
cd api
pnpm add @bull-board/api @bull-board/express

# Add to app.module.ts
import { createBullBoard } from '@bull-board/api';
import { ExpressAdapter } from '@bull-board/express';

const serverAdapter = new ExpressAdapter();
createBullBoard({
  queues: [webhookQueue],
  serverAdapter,
});

serverAdapter.setBasePath('/queues');
app.use('/queues', serverAdapter.getRouter());

# Open dashboard
open http://localhost:3000/queues
```

---

### 3. Test with HubSpot CLI

```bash
# Trigger test webhook
curl -X POST https://api.hubspot.com/webhooks/v3/{appId}/subscriptions/test \
  -H "Authorization: Bearer YOUR_DEVELOPER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subscriptionId": 123,
    "subscriptionType": "contact.propertyChange"
  }'
```

---

### 4. Monitor Failed Jobs

```bash
# View failed jobs in Redis
redis-cli
> KEYS bull:hubspot-webhook:failed
> LRANGE bull:hubspot-webhook:failed 0 10

# View job details
> GET bull:hubspot-webhook:12345
```

---

### 5. Check Production Logs (BetterStack)

```bash
# Go to BetterStack dashboard
open https://logs.betterstack.com

# Search for webhook errors
# Query: level:error AND message:"webhook"

# Common patterns to search:
# - "Invalid webhook signature"
# - "Webhook processing failed"
# - "Lead not found for HubSpot contact"
```

---

## Common Issues

### 1. Invalid Signature Error

**Error:**
```
UnauthorizedException: Invalid webhook signature
```

**Causes:**
- Wrong `HUBSPOT_WEBHOOK_SECRET` in .env
- Using `JSON.stringify(body)` instead of `rawBody`
- `rawBody: true` not set in `main.ts`

**Solution:**
```bash
# 1. Verify secret matches HubSpot app
cat api/.env | grep HUBSPOT_WEBHOOK_SECRET
# Compare with HubSpot Developer Dashboard → Webhooks → Webhook Secret

# 2. Verify rawBody enabled
cat api/src/main.ts | grep rawBody
# Should see: const app = await NestFactory.create(AppModule, { rawBody: true });

# 3. Restart API
cd api && pnpm start:dev
```

---

### 2. Webhook Timeout

**Error:**
```
[HubSpot] Webhook delivery failed: Timeout after 5 seconds
```

**Cause:** Slow synchronous processing in controller

**Solution:**
```typescript
// ❌ WRONG - Blocking
@Post('webhooks/hubspot')
async handleHubSpotWebhook(@Body() payload: any) {
  await this.processWebhook(payload); // Blocks for 10+ seconds
  return { received: true };
}

// ✅ CORRECT - Non-blocking
@Post('webhooks/hubspot')
async handleHubSpotWebhook(@Body() payload: any) {
  await this.webhookQueue.add('hubspot-event', payload); // Queues in <100ms
  return { received: true }; // Returns immediately
}
```

---

### 3. Lead Not Found

**Error:**
```
[HubSpotWebhookProcessor] Lead not found for HubSpot contact 12345
```

**Causes:**
- Contact not yet synced to Partner Connector
- Wrong HubSpot contact ID mapping

**Solution:**
```bash
# 1. Trigger manual sync to import contact
curl -X POST http://localhost:3000/v1/integrations/hubspot/sync \
  -H "Cookie: access_token=YOUR_JWT_TOKEN"

# 2. Verify lead exists in MongoDB
mongosh "your-mongodb-url"
use partner-connector
db.leads.findOne({ "hubspot_contact_id": 12345 })

# 3. If not found, check HubSpot API
curl -X GET "https://api.hubapi.com/crm/v3/objects/contacts/12345" \
  -H "Authorization: Bearer YOUR_HUBSPOT_TOKEN"
```

---

### 4. Duplicate Webhook Deliveries

**Issue:** Same webhook received multiple times

**Cause:** HubSpot retries if no 200 OK received within 5 seconds

**Solution:**
```typescript
// Implement idempotency key
@Post('webhooks/hubspot')
async handleHubSpotWebhook(@Body() payload: any) {
  const { eventId } = payload;

  // Check if already processed
  const existing = await this.eventsRepository.findOne({ eventId });
  if (existing) {
    return { received: true, duplicate: true };
  }

  // Store event ID
  await this.eventsRepository.create({ eventId, processedAt: new Date() });

  // Process webhook
  await this.webhookQueue.add('hubspot-event', payload);
  return { received: true };
}
```

---

### 5. BullMQ Job Stuck

**Issue:** Job remains in "active" state indefinitely

**Cause:** Worker crashed during processing

**Solution:**
```bash
# 1. Check active jobs
redis-cli
> LRANGE bull:hubspot-webhook:active 0 -1

# 2. Move stuck job to failed queue
> RPOPLPUSH bull:hubspot-webhook:active bull:hubspot-webhook:failed

# 3. Restart API to reprocess failed jobs
cd api && pnpm start:dev
```

---

### 6. High Memory Usage

**Issue:** Redis memory growing continuously

**Cause:** Completed jobs not removed

**Solution:**
```typescript
// In LeadsModule
BullModule.registerQueue({
  name: QUEUES.HUBSPOT_WEBHOOK,
  defaultJobOptions: {
    removeOnComplete: 100,  // ✅ Keep only last 100
    removeOnFail: 500,      // ✅ Keep only last 500
  },
})
```

---

## Next Steps

### For Developers
1. ✅ Review [OAuth Integration Guide](./OAUTH_INTEGRATION_GUIDE.md) - How OAuth connects partners
2. ✅ Review [Bidirectional Sync Guide](https://github.com/partner-connector/api/blob/main/docs/SPRINT_11_PHASE_6_COMPLETE.md) - Two-way sync
3. ✅ Review [ARCHITECTURE.md](./ARCHITECTURE.md) - Full system architecture

### For Troubleshooting
1. Check webhook logs: `tail -f api/logs/combined.log | grep webhook`
2. Check BullMQ dashboard: `http://localhost:3000/queues`
3. Check Redis: `redis-cli KEYS bull:hubspot-webhook:*`
4. Check BetterStack: [logs.betterstack.com](https://logs.betterstack.com)

---

**Last Updated:** January 28, 2026 | **Next Review:** February 28, 2026
