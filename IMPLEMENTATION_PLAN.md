# Partner Connector: Comprehensive Feature Implementation Plan

## Executive Summary

**Timeline:** 8-12 weeks across 6 sprints
**Complexity:** High - Full-stack development with external integrations
**Repositories:** Both `api-main` (NestJS backend) and `client-main` (Nuxt 4 frontend)

### Key Integrations
- **Stripe:** Full payment automation (invoicing, webhooks, manual controls)
- **Mailgun:** Email notifications (reminders, alerts, password reset)
- **ChiliPiper:** Meeting data sync to leads
- **Cron Jobs:** Scheduled tasks (daily reminders, monthly invoices)

### User Decisions (Confirmed)
✅ **Scheduled Date:** Pull most recent meeting (any status)
✅ **Reminder Emails:** Include both summary and magic link
✅ **DB Migration:** Simple connection string update
✅ **Stripe Scope:** Full automation + admin controls

---

## Sprint Overview

| Sprint | Duration | Focus Area | Repository |
|--------|----------|------------|------------|
| **Sprint 1** | 2 weeks | Data Foundation & Sync | api-main |
| **Sprint 2** | 3 weeks | Email & Notifications | Both |
| **Sprint 3** | 2 weeks | User Management & Admin | Both |
| **Sprint 4** | 3 weeks | Stripe Integration | Both |
| **Sprint 5** | 2 weeks | Frontend Pages & UX | client-main |
| **Sprint 6** | 1 week | Bug Fixes & Cleanup | Both |

---

## SPRINT 1: Data Foundation & Core Sync (2 weeks)

### Goal
Enable proper data relationships between leads and meetings. Foundation for all other features.

### 1.1 Add `scheduled_date` Field to Lead Schema

**Complexity:** Medium
**Repository:** `api-main`

**Files to Modify:**
```
api-main/src/modules/leads/schemas/lead.schema.ts
api-main/src/modules/leads/types/leads.types.ts
api-main/src/modules/leads/dto/lead.response.dto.ts
```

**Implementation:**
```typescript
// In lead.schema.ts
@Prop({ type: Date, default: null, index: true })
scheduledDate: Date | null;
```

**Database Considerations:**
- No explicit migration needed (MongoDB handles new fields gracefully)
- Add index for performance: `LeadSchema.index({ scheduledDate: 1 })`
- Backfill existing leads: Query meetings for each lead, update with most recent `scheduledTime`

---

### 1.2 Real-Time Meeting → Lead Sync

**Complexity:** Medium
**Repository:** `api-main`

**Files to Modify:**
```
api-main/src/modules/meetings/meetings.service.ts
api-main/src/modules/leads/leads.service.ts
```

**Implementation Logic:**
1. When ChiliPiper webhook processed (meeting created/updated):
   - Find or create lead
   - Query all meetings for that lead
   - Find most recent meeting (by `scheduledTime`)
   - Update lead's `scheduledDate` field

**Code Example:**
```typescript
// In MeetingsService.processMeetingUpdate()
async processMeetingUpdate(payload) {
  // ... existing code to create/update meeting ...

  // After meeting created/updated:
  const latestMeeting = await this._meetingsRepository.findOne(
    { lead: lead._id },
    { sort: { scheduledTime: -1 } }
  );

  if (latestMeeting) {
    await this._leadsService.updateById(lead._id, {
      scheduledDate: latestMeeting.scheduledTime
    });
  }
}
```

**Edge Cases:**
- Multiple meetings: Sort by `scheduledTime DESC`, take first
- Cancelled meetings: Still include (requirement: "any status")
- No meetings: `scheduledDate` remains `null`

---

### 1.3 Database Migration

**Complexity:** Simple
**Repository:** `api-main`

**Files to Modify:**
```
api-main/.env.dev
api-main/.env.production
```

**Steps:**
1. **Backup current database** via MongoDB Atlas
2. Create new dedicated MongoDB instance
3. Export data from shared DB: `mongodump --uri="old-connection-string"`
4. Import to new DB: `mongorestore --uri="new-connection-string"`
5. Update `.env.dev` and `.env.production`:
   ```env
   MONGODB_URL=mongodb+srv://new-dedicated-instance
   ```
6. Test connection: `pnpm start:dev`
7. Monitor logs for errors
8. Run validation queries to ensure data integrity

**Rollback Plan:** Keep old connection string in backup file, revert if issues

---

### 1.4 Lead Database Cleanup

**Complexity:** Low
**Repository:** `api-main`

**Tasks:**
1. Remove duplicate leads (by email)
2. Normalize email addresses (lowercase)
3. Remove test/invalid data
4. Ensure all indexes exist

**Cleanup Script:**
```typescript
// Run as one-time script via NestJS command
async function cleanupLeads() {
  // Remove duplicates (keep oldest)
  const duplicates = await Lead.aggregate([
    { $group: { _id: '$email', count: { $sum: 1 }, ids: { $push: '$_id' } } },
    { $match: { count: { $gt: 1 } } }
  ]);

  for (const dup of duplicates) {
    const [keep, ...remove] = dup.ids;
    await Lead.deleteMany({ _id: { $in: remove } });
  }

  // Normalize emails
  await Lead.updateMany({}, [
    { $set: { email: { $toLower: '$email' } } }
  ]);

  // Ensure indexes
  await Lead.collection.createIndex({ email: 1 }, { unique: true });
  await Lead.collection.createIndex({ scheduledDate: 1 });
  await Lead.collection.createIndex({ partner: 1 });
}
```

---

## SPRINT 2: Email & Notification System (3 weeks)

### Goal
Implement email infrastructure, password reset, and automated reminder system.

### 2.1 Mailgun Email Service Wrapper

**Complexity:** Medium
**Repository:** `api-main`

**New Files:**
```
api-main/src/common/mailgun/mailgun.service.ts (NEW)
api-main/src/common/mailgun/templates/ (NEW directory)
api-main/src/common/mailgun/templates/lead-reminder.hbs (NEW)
api-main/src/common/mailgun/templates/password-reset.hbs (NEW)
api-main/src/common/mailgun/templates/lost-lead-alert.hbs (NEW)
```

**Implementation:**
```typescript
@Injectable()
export class MailgunService {
  constructor(private readonly mailgunHelper: MailgunHelper) {}

  async sendLeadStatusReminder(params: {
    userEmail: string;
    userName: string;
    overdueSummary: Array<{
      email: string;
      company: string;
      daysSinceUpdate: number;
    }>;
    magicLink: string;
  }) {
    const html = this.renderTemplate('lead-reminder', params);

    await this.mailgunHelper.send(process.env.MAILGUN_DOMAIN, {
      from: 'Partner Connector <noreply@belkins.io>',
      to: params.userEmail,
      subject: `Action Required: ${params.overdueSummary.length} leads need status updates`,
      html
    });
  }

  async sendPasswordReset(params: { email: string; resetToken: string }) {
    const resetLink = `${process.env.CLIENT_URL}/reset-password?token=${params.resetToken}`;
    const html = this.renderTemplate('password-reset', { resetLink });

    await this.mailgunHelper.send(process.env.MAILGUN_DOMAIN, {
      from: 'Partner Connector <noreply@belkins.io>',
      to: params.email,
      subject: 'Password Reset Request',
      html
    });
  }

  async sendLostLeadAlert(lead: Lead) {
    const html = this.renderTemplate('lost-lead-alert', {
      email: lead.email,
      status: lead.dealStatus,
      timestamp: new Date().toISOString()
    });

    await this.mailgunHelper.send(process.env.MAILGUN_DOMAIN, {
      from: 'Partner Connector <noreply@belkins.io>',
      to: 'julia@belkins.io',
      subject: `Lost Lead Alert: ${lead.email}`,
      html
    });
  }

  private renderTemplate(name: string, data: any): string {
    // Use handlebars or similar
    const template = fs.readFileSync(`templates/${name}.hbs`, 'utf8');
    return handlebars.compile(template)(data);
  }
}
```

**Environment Variables:**
```env
MAILGUN_DOMAIN=mg.belkins.io
MAILGUN_API_KEY=key-xxx
CLIENT_URL=http://localhost:8000
```

---

### 2.2 Password Reset Flow

**Complexity:** Medium
**Repository:** Both

**Backend Files:**
```
api-main/src/modules/users/schemas/users.schema.ts
api-main/src/modules/auth/auth.service.ts
api-main/src/modules/auth/auth.controller.ts
api-main/src/modules/auth/dto/password-reset.dto.ts (NEW)
```

**Frontend Files:**
```
client-main/app/pages/forgot-password.vue (NEW)
client-main/app/pages/reset-password.vue (NEW)
client-main/app/gateway/auth/auth.gateway.ts
```

**Backend Schema Changes:**
```typescript
// Add to User schema
@Prop({ type: String, default: null })
resetToken: string | null;

@Prop({ type: Date, default: null })
resetTokenExpiry: Date | null;
```

**Backend Endpoints:**
```typescript
// POST /auth/forgot-password
async requestPasswordReset(@Body() dto: ForgotPasswordDto) {
  const user = await this.usersRepository.findOne({ email: dto.email });
  if (!user) return { message: 'If email exists, reset link sent' }; // Prevent enumeration

  const resetToken = crypto.randomBytes(32).toString('hex');
  const expiry = new Date(Date.now() + 3600000); // 1 hour

  await this.usersRepository.updateById(user._id, {
    resetToken,
    resetTokenExpiry: expiry
  });

  await this.mailgunService.sendPasswordReset({
    email: user.email,
    resetToken
  });

  return { message: 'If email exists, reset link sent' };
}

// POST /auth/reset-password
async resetPassword(@Body() dto: ResetPasswordDto) {
  const user = await this.usersRepository.findOne({
    resetToken: dto.token,
    resetTokenExpiry: { $gt: new Date() }
  });

  if (!user) throw new UnauthorizedException('Invalid or expired token');

  const hashedPassword = await bcrypt.hash(dto.newPassword, 10);

  await this.usersRepository.updateById(user._id, {
    password: hashedPassword,
    resetToken: null,
    resetTokenExpiry: null
  });

  // Blacklist all existing tokens for this user
  await this.tokenRepository.blacklistUserTokens(user._id);

  return { message: 'Password reset successful' };
}
```

**Frontend Pages:**
```vue
<!-- forgot-password.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { AuthGateway } from '~/gateway/auth'

const email = ref('')
const submitted = ref(false)

const handleSubmit = async () => {
  await AuthGateway.requestPasswordReset({ email: email.value })
  submitted.value = true
}
</script>

<template>
  <div class="max-w-md mx-auto mt-20">
    <Card>
      <CardHeader>
        <CardTitle>Forgot Password</CardTitle>
        <CardDescription>Enter your email to receive a reset link</CardDescription>
      </CardHeader>
      <CardContent>
        <div v-if="!submitted">
          <Input v-model="email" type="email" placeholder="Email" />
          <Button @click="handleSubmit" class="mt-4">Send Reset Link</Button>
        </div>
        <div v-else>
          <p>If an account exists with that email, you'll receive a reset link.</p>
        </div>
      </CardContent>
    </Card>
  </div>
</template>
```

---

### 2.3 Daily Lead Status Reminder Cron Job

**Complexity:** High
**Repository:** `api-main`

**New Files:**
```
api-main/src/modules/notifications/ (NEW module)
api-main/src/modules/notifications/notifications.module.ts (NEW)
api-main/src/modules/notifications/notifications.service.ts (NEW)
api-main/src/modules/notifications/notifications.cron.ts (NEW)
api-main/src/app.module.ts (import ScheduleModule)
```

**Setup:**
```bash
cd api-main
pnpm add @nestjs/schedule
```

**Implementation:**
```typescript
// app.module.ts
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(),
    // ... other modules
    NotificationsModule
  ]
})

// notifications.cron.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class NotificationsCron {
  private readonly logger = new Logger(NotificationsCron.name);

  constructor(
    private readonly notificationsService: NotificationsService,
    private readonly leadsService: LeadsService,
    private readonly usersService: UsersService,
    private readonly partnersService: PartnersService,
    private readonly mailgunService: MailgunService
  ) {}

  @Cron('0 9 * * *', { timeZone: 'America/Los_Angeles' })
  async sendDailyLeadReminders() {
    this.logger.log('Starting daily lead status reminders');

    try {
      // Get all partners
      const partners = await this.partnersService.getMany();

      for (const partner of partners) {
        // Get users assigned to this partner
        const users = await this.usersService.getManyByPartner(partner._id);

        if (users.length === 0) continue;

        // Find leads needing status updates
        // Logic: scheduledDate exists AND scheduledDate < NOW AND updatedAt > 24 hours ago
        const overdueLeads = await this.leadsService.getMany({
          filter: [
            { column: 'partner', operator: 'eq', value: partner._id },
            { column: 'scheduledDate', operator: 'is_not_null' },
            { column: 'scheduledDate', operator: 'lt', value: new Date() },
            { column: 'dealStatus', operator: 'ne', value: 'won' },
            { column: 'dealStatus', operator: 'ne', value: 'lost_no_show' },
            { column: 'dealStatus', operator: 'ne', value: 'lost_disqualified' },
            { column: 'dealStatus', operator: 'ne', value: 'lost_no_potential' }
          ]
        });

        if (overdueLeads.length === 0) continue;

        // Generate magic link
        const magicLink = this.notificationsService.generateMagicLink({
          partnerId: partner._id.toString(),
          filter: 'overdue'
        });

        // Send email to all partner users
        for (const user of users) {
          await this.mailgunService.sendLeadStatusReminder({
            userEmail: user.email,
            userName: user.name,
            overdueSummary: overdueLeads.list.map(lead => ({
              email: lead.email,
              company: lead.company || 'N/A',
              daysSinceUpdate: this.calculateDaysSince(lead.updatedAt)
            })),
            magicLink
          });
        }

        this.logger.log(`Sent reminders for ${partner.name}: ${overdueLeads.length} overdue leads`);
      }

      this.logger.log('Completed daily lead status reminders');
    } catch (error) {
      this.logger.error('Failed to send daily reminders', error.stack);
    }
  }

  private calculateDaysSince(date: Date): number {
    const now = new Date();
    const diffMs = now.getTime() - new Date(date).getTime();
    return Math.floor(diffMs / (1000 * 60 * 60 * 24));
  }
}

// notifications.service.ts
@Injectable()
export class NotificationsService {
  constructor(
    @Inject(JWT_SERVICE) private readonly jwtService: JwtService,
    @Inject(CONFIG_SERVICE) private readonly configService: ConfigService
  ) {}

  generateMagicLink(params: {
    partnerId: string;
    filter: string;
  }): string {
    const token = this.jwtService.sign({
      partnerId: params.partnerId,
      filter: params.filter,
      exp: Math.floor(Date.now() / 1000) + 86400 // 24 hours
    }, {
      secret: this.configService.get('MAGIC_LINK_SECRET')
    });

    const clientUrl = this.configService.get('CLIENT_URL');
    return `${clientUrl}/leads?magic=${token}`;
  }

  async validateMagicLink(token: string) {
    try {
      return this.jwtService.verify(token, {
        secret: this.configService.get('MAGIC_LINK_SECRET')
      });
    } catch {
      throw new UnauthorizedException('Invalid or expired magic link');
    }
  }
}
```

**Frontend Magic Link Handler:**
```vue
<!-- client-main/app/pages/leads/index.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { LeadsGateway } from '~/gateway/leads'

const route = useRoute()

onMounted(async () => {
  // Check for magic link
  if (route.query.magic) {
    try {
      const decoded = await LeadsGateway.validateMagicLink(route.query.magic as string)

      // Apply filter for overdue leads
      if (decoded.filter === 'overdue') {
        applyFilter([
          { column: 'scheduledDate', operator: 'is_not_null' },
          { column: 'scheduledDate', operator: 'lt', value: new Date() },
          { column: 'dealStatus', operator: 'not_in', value: ['won', 'lost_no_show', 'lost_disqualified', 'lost_no_potential'] }
        ])
      }

      // Remove magic param from URL
      router.replace({ query: {} })
    } catch (error) {
      console.error('Invalid magic link', error)
    }
  }
})
</script>
```

**Environment Variables:**
```env
CRON_TIMEZONE=America/Los_Angeles
MAGIC_LINK_SECRET=random-secret-generate-me
```

---

### 2.4 Lost Lead Email Alerts

**Complexity:** Medium
**Repository:** `api-main`

**Files to Modify:**
```
api-main/src/modules/leads/leads.service.ts
api-main/src/modules/notifications/notifications.service.ts
```

**Implementation:**
```typescript
// In LeadsService - whenever dealStatus changes
async update(filter: any, update: Partial<Lead>) {
  const oldLead = await this.repository.getOne(filter);
  const updatedLead = await this.repository.update(filter, update);

  // Check if status changed to "lost"
  const lostStatuses = ['lost_no_show', 'lost_disqualified', 'lost_no_potential'];
  if (
    update.dealStatus &&
    lostStatuses.includes(update.dealStatus) &&
    oldLead.dealStatus !== update.dealStatus
  ) {
    await this.notificationsService.sendLostLeadAlert(updatedLead);
  }

  return updatedLead;
}

// In NotificationsService
async sendLostLeadAlert(lead: Lead) {
  await this.mailgunService.sendLostLeadAlert(lead);
}
```

---

## SPRINT 3: User Management & Admin Features (2 weeks)

### Goal
Enhance admin controls for user management and partner configuration.

### 3.1 User Active/Inactive Status

**Complexity:** Medium
**Repository:** Both

**Backend Files:**
```
api-main/src/modules/users/schemas/users.schema.ts
api-main/src/modules/users/users.service.ts
api-main/src/modules/users/users.controller.ts
api-main/src/modules/users/dto/update-user.dto.ts
api-main/src/modules/auth/auth.service.ts
api-main/src/common/guards/auth.guard.ts
```

**Frontend Files:**
```
client-main/app/pages/users.vue
client-main/app/components/shadcn-ui/switch (use existing)
```

**Backend Implementation:**
```typescript
// In User schema
@Prop({ type: Boolean, default: true, index: true })
isActive: boolean;

// In AuthService - block inactive users on login
async signIn(email: string, password: string) {
  const user = await this.usersRepository.getWithPassword({
    email,
    isActive: true  // ← Only allow active users
  });

  if (!user) {
    throw new UnauthorizedException('Invalid credentials or account inactive');
  }

  // ... rest of sign-in logic
}

// In AuthGuard - block inactive users on requests
async canActivate(context: ExecutionContext) {
  const request = context.switchToHttp().getRequest();
  const { accessToken } = this.cookieUtils.getTokensFromRequest(request);

  const decoded = this.jwtService.verify(accessToken);
  const user = await this.usersRepository.getOne({
    _id: decoded.userId,
    isActive: true  // ← Only allow active users
  });

  if (!user) {
    throw new UnauthorizedException('Account is inactive');
  }

  // ... rest of guard logic
}

// In UsersController - add toggle endpoint
@Patch(':id/status')
@RouteGuard({ role: IUsers.Enum.Role.Admin })
async toggleStatus(
  @Param('id', ParseMongoIdPipe) id: Types.ObjectId,
  @Body() dto: ToggleStatusDto
) {
  return this.usersService.updateById(id, { isActive: dto.isActive });
}
```

**Frontend Implementation:**
```vue
<!-- users.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { UsersGateway } from '~/gateway/users'
import { Switch } from '~/components/shadcn-ui/switch'

const users = ref([])

const toggleActive = async (userId: string, isActive: boolean) => {
  await UsersGateway.toggleStatus({ id: userId, isActive })
  // Refresh users list
  await fetchUsers()
}
</script>

<template>
  <Table>
    <TableBody>
      <TableRow v-for="user in users" :key="user._id">
        <TableCell>{{ user.name }}</TableCell>
        <TableCell>{{ user.email }}</TableCell>
        <TableCell>
          <div class="flex items-center gap-2">
            <Switch
              :model-value="user.isActive"
              @update:model-value="toggleActive(user._id, $event)"
            />
            <span :class="user.isActive ? 'text-green-600' : 'text-red-600'">
              {{ user.isActive ? 'Active' : 'Inactive' }}
            </span>
          </div>
        </TableCell>
      </TableRow>
    </TableBody>
  </Table>
</template>
```

**Session Revocation:**
When user is deactivated:
1. AuthGuard blocks further requests (checks `isActive`)
2. User's existing session becomes invalid on next request
3. No need for manual token blacklisting

---

### 3.2 Partner ICP Editing UI

**Complexity:** Simple
**Repository:** `client-main`

**Files:**
```
client-main/app/pages/partners/[slug]/settings.vue
client-main/app/components/pages/partners/PartnerICPEditor.vue (NEW)
client-main/app/gateway/partners/partners.gateway.ts
```

**Implementation:**
```vue
<!-- PartnerICPEditor.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { PartnersGateway } from '~/gateway/partners'
import { Button } from '~/components/shadcn-ui/button'
import { Input } from '~/components/shadcn-ui/input'
import { Card, CardContent, CardHeader, CardTitle } from '~/components/shadcn-ui/card'

const props = defineProps<{
  partnerId: string
  initialIcp: Record<string, string> | null
}>()

const emit = defineEmits<{
  saved: []
}>()

const icp = ref<Record<string, string>>(props.initialIcp || {})
const newKey = ref('')
const newValue = ref('')

const addField = () => {
  if (!newKey.value) return
  icp.value[newKey.value] = newValue.value
  newKey.value = ''
  newValue.value = ''
}

const removeField = (key: string) => {
  delete icp.value[key]
}

const save = async () => {
  await PartnersGateway.update({
    id: props.partnerId,
    icp: icp.value
  })
  emit('saved')
}
</script>

<template>
  <Card>
    <CardHeader>
      <CardTitle>Ideal Customer Profile (ICP)</CardTitle>
    </CardHeader>
    <CardContent class="space-y-4">
      <!-- Existing fields -->
      <div v-for="(value, key) in icp" :key="key" class="flex gap-2">
        <Input :model-value="key" disabled class="w-1/3" />
        <Input v-model="icp[key]" class="flex-1" />
        <Button @click="removeField(key)" variant="outline" size="icon">
          <Trash class="h-4 w-4" />
        </Button>
      </div>

      <!-- Add new field -->
      <div class="flex gap-2 pt-4 border-t">
        <Input v-model="newKey" placeholder="Field name" class="w-1/3" />
        <Input v-model="newValue" placeholder="Value" class="flex-1" />
        <Button @click="addField" variant="outline">
          <Plus class="h-4 w-4 mr-2" />
          Add
        </Button>
      </div>

      <Button @click="save" class="w-full">
        Save ICP
      </Button>
    </CardContent>
  </Card>
</template>
```

**Backend:**
Partner schema already has `icp: Record<string, string>`. Just needs UPDATE endpoint:

```typescript
// In PartnersController
@Patch(':id')
@RouteGuard({ role: IUsers.Enum.Role.Admin })
async update(
  @Param('id', ParseMongoIdPipe) id: Types.ObjectId,
  @Body() dto: UpdatePartnerDto
) {
  return this.partnersService.updateById(id, dto);
}
```

---

### 3.3 Admin Table Enhancements

**Complexity:** Medium
**Repository:** Both

**Backend Enhancement:**
```
api-main/src/modules/partners/partners.service.ts
api-main/src/modules/partners/partners.controller.ts
```

**Frontend Enhancement:**
```
client-main/app/pages/partners/index.vue
client-main/app/components/sections/partners-table/PartnersTable.vue
```

**Implementation:**

**Backend - Add stats to partner response:**
```typescript
// In PartnersService
async getManyWithStats() {
  const partners = await this.repository.getMany();

  // Aggregate last appointment per partner
  const appointmentStats = await this.meetingsRepository.aggregate([
    {
      $lookup: {
        from: 'leads',
        localField: 'lead',
        foreignField: '_id',
        as: 'leadData'
      }
    },
    { $unwind: '$leadData' },
    {
      $group: {
        _id: '$leadData.partner',
        lastAppointmentDate: { $max: '$scheduledTime' },
        totalAppointments: { $sum: 1 }
      }
    }
  ]);

  // Merge stats into partners
  return partners.map(p => {
    const stats = appointmentStats.find(s => s._id.equals(p._id));
    return {
      ...p.toObject(),
      lastAppointmentDate: stats?.lastAppointmentDate || null,
      totalAppointments: stats?.totalAppointments || 0
    };
  });
}
```

**Frontend - Add columns:**
```typescript
// In PartnersTable.vue columns configuration
const columns = [
  { key: 'name', label: 'Partner Name' },
  { key: 'icp', label: 'ICP', render: (row) => renderICP(row.icp) },
  { key: 'lastAppointmentDate', label: 'Last Appointment', render: (row) => formatDate(row.lastAppointmentDate) },
  { key: 'totalAppointments', label: 'Total Meetings' },
  { key: 'createdAt', label: 'Created' }
]

function renderICP(icp: Record<string, string> | null) {
  if (!icp) return '-'
  return Object.entries(icp)
    .map(([k, v]) => `${k}: ${v}`)
    .join(', ')
}
```

---

### 3.4 Partner Column Visibility (Admin-Only)

**Complexity:** Medium
**Repository:** Both

**Backend:**
```
api-main/src/modules/leads/leads.service.ts
```

**Frontend:**
```
client-main/app/components/sections/leads-table/LeadsDataTable.vue
client-main/app/components/sections/leads-table/table.ts
```

**Implementation:**

**Backend - Conditional population:**
```typescript
// In LeadsService.getMany()
async getMany(query: GetManyDto, user: User) {
  const leads = await this.repository.getMany(query);

  // Only populate partner for admin/owner
  if (user.role === 'admin' || user.role === 'owner') {
    return await this.repository.populate(leads, 'partner');
  }

  return leads;
}
```

**Frontend - Conditional column:**
```typescript
// In table.ts
export const getColumns = (userRole: string) => {
  const baseColumns = [
    { key: 'email', label: 'Email' },
    { key: 'company', label: 'Company' },
    { key: 'dealStatus', label: 'Status' },
    { key: 'scheduledDate', label: 'Scheduled' }
  ]

  // Add Partner column for admin/owner only
  if (userRole === 'admin' || userRole === 'owner') {
    baseColumns.splice(1, 0, {
      key: 'partner',
      label: 'Partner',
      render: (row) => row.partner?.name || '-'
    })
  }

  return baseColumns
}
```

---

## SPRINT 4: Stripe Integration & Finance (3 weeks)

### Goal
Full Stripe payment automation with admin controls.

### 4.1 Stripe Setup & Service

**Complexity:** High
**Repository:** `api-main`

**Installation:**
```bash
cd api-main
pnpm add stripe
```

**New Files:**
```
api-main/src/common/stripe/stripe.service.ts (NEW)
api-main/src/common/stripe/stripe.module.ts (NEW)
api-main/src/modules/billing/billing.cron.ts (NEW)
api-main/src/modules/billing/invoice.service.ts (NEW)
api-main/src/modules/billing/stripe-webhook.processor.ts (NEW)
```

**Environment Variables:**
```env
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
```

**Implementation:**
```typescript
// stripe.service.ts
import Stripe from 'stripe';

@Injectable()
export class StripeService {
  private stripe: Stripe;

  constructor(private configService: ConfigService) {
    this.stripe = new Stripe(
      this.configService.get('STRIPE_SECRET_KEY'),
      { apiVersion: '2024-11-20.acacia' }
    );
  }

  async createCustomer(params: {
    email: string;
    name: string;
    metadata: Record<string, string>;
  }) {
    return this.stripe.customers.create({
      email: params.email,
      name: params.name,
      metadata: params.metadata
    });
  }

  async createInvoice(params: {
    customerId: string;
    amount: number;
    description: string;
    metadata: Record<string, string>;
  }) {
    const invoice = await this.stripe.invoices.create({
      customer: params.customerId,
      auto_advance: false, // Manual finalization
      metadata: params.metadata
    });

    await this.stripe.invoiceItems.create({
      customer: params.customerId,
      invoice: invoice.id,
      amount: params.amount,
      currency: 'usd',
      description: params.description
    });

    return invoice;
  }

  async finalizeInvoice(invoiceId: string) {
    return this.stripe.invoices.finalizeInvoice(invoiceId, {
      auto_advance: true // Auto-send to customer
    });
  }

  async voidInvoice(invoiceId: string) {
    return this.stripe.invoices.voidInvoice(invoiceId);
  }

  async manualCharge(customerId: string, amount: number, description: string) {
    return this.stripe.paymentIntents.create({
      customer: customerId,
      amount,
      currency: 'usd',
      description,
      automatic_payment_methods: { enabled: true }
    });
  }

  async addCredit(customerId: string, amount: number, description: string) {
    return this.stripe.customerBalanceTransactions.create({
      customer: customerId,
      amount: -amount, // Negative = credit
      currency: 'usd',
      description
    });
  }

  constructEvent(rawBody: Buffer, signature: string, secret: string) {
    return this.stripe.webhooks.constructEvent(rawBody, signature, secret);
  }
}
```

---

### 4.2 Monthly Invoice Generation Cron

**Complexity:** High
**Repository:** `api-main`

**Files:**
```
api-main/src/modules/billing/billing.cron.ts (NEW)
api-main/src/modules/billing/billing.service.ts
api-main/src/modules/billing/invoice.service.ts (NEW)
```

**Implementation:**
```typescript
// billing.cron.ts
@Injectable()
export class BillingCron {
  private readonly logger = new Logger(BillingCron.name);

  constructor(
    private readonly billingService: BillingService,
    private readonly invoiceService: InvoiceService,
    private readonly leadsService: LeadsService,
    private readonly partnersService: PartnersService,
    private readonly stripeService: StripeService
  ) {}

  @Cron('0 0 1 * *') // 1st of every month at midnight
  async generateMonthlyInvoices() {
    this.logger.log('Starting monthly invoice generation');

    const billingPeriod = moment().subtract(1, 'month').format('YYYY-MM');
    const partners = await this.partnersService.getManyWithBilling();

    for (const partner of partners) {
      try {
        if (!partner.billing) {
          this.logger.warn(`Partner ${partner.name} has no billing config`);
          continue;
        }

        // Get all leads for this partner in billing period
        const leads = await this.leadsService.getMany({
          filter: [
            { column: 'partner', operator: 'eq', value: partner._id },
            { column: 'createdAt', operator: 'gte', value: moment(billingPeriod).startOf('month').toDate() },
            { column: 'createdAt', operator: 'lte', value: moment(billingPeriod).endOf('month').toDate() }
          ]
        });

        // Calculate amount based on billing terms
        const amount = this.billingService.calculateMonthlyCharge({
          billing: partner.billing,
          leads: leads.list,
          month: billingPeriod
        });

        if (amount === 0) {
          this.logger.log(`No charge for ${partner.name} - amount is 0`);
          continue;
        }

        // Ensure partner has Stripe customer ID
        if (!partner.stripeCustomerId) {
          const customer = await this.stripeService.createCustomer({
            email: partner.email || 'billing@example.com',
            name: partner.name,
            metadata: { partnerId: partner._id.toString() }
          });

          await this.partnersService.updateById(partner._id, {
            stripeCustomerId: customer.id
          });

          partner.stripeCustomerId = customer.id;
        }

        // Create Stripe invoice
        const stripeInvoice = await this.stripeService.createInvoice({
          customerId: partner.stripeCustomerId,
          amount: amount * 100, // Convert to cents
          description: `Partner Connector - ${billingPeriod}`,
          metadata: {
            partnerId: partner._id.toString(),
            billingPeriod
          }
        });

        // Save to database
        const invoice = await this.invoiceService.create({
          stripeInvoiceId: stripeInvoice.id,
          partner: partner._id,
          billing: partner.billing._id,
          amount,
          stripeStatus: 'draft',
          billingPeriod
        });

        // Create invoice lines for each lead
        for (const lead of leads.list) {
          const leadCharge = this.billingService.calculateLeadCharge(partner.billing, lead);
          if (leadCharge > 0) {
            await this.invoiceService.createLine({
              invoice: invoice._id,
              lead: lead._id,
              billing: partner.billing._id,
              billingPeriod,
              amount: leadCharge
            });
          }
        }

        // Finalize and send invoice
        await this.stripeService.finalizeInvoice(stripeInvoice.id);

        // Update status
        await this.invoiceService.updateById(invoice._id, {
          stripeStatus: 'open'
        });

        this.logger.log(`Generated invoice for ${partner.name}: $${amount}`);

      } catch (error) {
        this.logger.error(`Failed to generate invoice for ${partner.name}`, error.stack);
      }
    }

    this.logger.log('Completed monthly invoice generation');
  }
}

// billing.service.ts
calculateMonthlyCharge(params: {
  billing: Billing;
  leads: Lead[];
  month: string;
}): number {
  const monthNum = moment(params.month).month() + 1;

  // Find applicable billing term
  const term = params.billing.billingTerms.find(t =>
    t.startMonth <= monthNum &&
    (!t.endMonth || t.endMonth >= monthNum)
  );

  if (!term) return 0;

  // Calculate based on billing type
  switch (params.billing.billingType) {
    case 'qualified_lead':
      return params.leads.length * term.value;

    case 'closed_won_lead':
      const wonLeads = params.leads.filter(l => l.dealStatus === 'won');
      return wonLeads.length * term.value;

    case 'percentage_of_deal':
      const totalDealValue = params.leads
        .filter(l => l.dealStatus === 'won')
        .reduce((sum, l) => sum + (l.dealAmount || 0), 0);
      return totalDealValue * (term.value / 100);

    default:
      return 0;
  }
}

calculateLeadCharge(billing: Billing, lead: Lead): number {
  const monthNum = moment().month() + 1;
  const term = billing.billingTerms.find(t =>
    t.startMonth <= monthNum &&
    (!t.endMonth || t.endMonth >= monthNum)
  );

  if (!term) return 0;

  switch (billing.billingType) {
    case 'qualified_lead':
      return term.value;

    case 'closed_won_lead':
      return lead.dealStatus === 'won' ? term.value : 0;

    case 'percentage_of_deal':
      return lead.dealStatus === 'won'
        ? (lead.dealAmount || 0) * (term.value / 100)
        : 0;

    default:
      return 0;
  }
}
```

**Partner Schema Update:**
```typescript
// Add to Partner schema
@Prop({ type: String, default: null })
stripeCustomerId: string | null;
```

---

### 4.3 Stripe Webhook Handler

**Complexity:** High
**Repository:** `api-main`

**Files:**
```
api-main/src/modules/billing/billing.controller.ts
api-main/src/modules/billing/stripe-webhook.processor.ts (NEW)
api-main/src/static/queues.ts
```

**Queue Setup:**
```typescript
// Add to queues.ts
STRIPE_WEBHOOK_QUEUE: 'stripe-webhook',
```

**Implementation:**
```typescript
// billing.controller.ts
@Post('webhooks/stripe')
@HttpCode(200)
async handleStripeWebhook(
  @Req() req: RawBodyRequest<Request>,
  @Headers('stripe-signature') signature: string
) {
  try {
    const event = this.stripeService.constructEvent(
      req.rawBody,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET
    );

    // Push to queue for async processing
    await this.stripeWebhookQueue.add('stripe-event', { event });

    return { received: true };
  } catch (error) {
    this.logger.error('Webhook signature verification failed', error);
    throw new BadRequestException('Invalid signature');
  }
}

// stripe-webhook.processor.ts
@Processor(QUEUES.STRIPE_WEBHOOK_QUEUE)
export class StripeWebhookProcessor extends WorkerHost {
  private readonly logger = new Logger(StripeWebhookProcessor.name);

  constructor(
    private readonly invoiceService: InvoiceService,
    private readonly mailgunService: MailgunService
  ) {
    super();
  }

  async process(job: Job<{ event: Stripe.Event }>) {
    const { event } = job.data;

    this.logger.log(`Processing Stripe event: ${event.type}`);

    try {
      switch (event.type) {
        case 'invoice.paid':
          await this.handleInvoicePaid(event.data.object as Stripe.Invoice);
          break;

        case 'invoice.payment_failed':
          await this.handlePaymentFailed(event.data.object as Stripe.Invoice);
          break;

        case 'invoice.voided':
          await this.handleInvoiceVoided(event.data.object as Stripe.Invoice);
          break;

        default:
          this.logger.log(`Unhandled event type: ${event.type}`);
      }
    } catch (error) {
      this.logger.error(`Failed to process event ${event.type}`, error.stack);
      throw error; // Re-throw for BullMQ retry
    }
  }

  private async handleInvoicePaid(invoice: Stripe.Invoice) {
    await this.invoiceService.updateByStripeId(invoice.id, {
      stripeStatus: 'Paid',
      paymentDate: new Date(invoice.status_transitions.paid_at! * 1000)
    });

    this.logger.log(`Invoice ${invoice.id} marked as paid`);
  }

  private async handlePaymentFailed(invoice: Stripe.Invoice) {
    await this.invoiceService.updateByStripeId(invoice.id, {
      stripeStatus: 'Failed'
    });

    // Send alert email
    const dbInvoice = await this.invoiceService.getOne({
      stripeInvoiceId: invoice.id
    });

    await this.mailgunService.send(process.env.MAILGUN_DOMAIN, {
      from: 'Partner Connector <noreply@belkins.io>',
      to: 'julia@belkins.io',
      subject: `Payment Failed: Invoice ${invoice.number}`,
      html: `Payment failed for invoice ${invoice.number}. Amount: $${invoice.amount_due / 100}`
    });

    this.logger.warn(`Invoice ${invoice.id} payment failed`);
  }

  private async handleInvoiceVoided(invoice: Stripe.Invoice) {
    await this.invoiceService.updateByStripeId(invoice.id, {
      stripeStatus: 'Void'
    });

    this.logger.log(`Invoice ${invoice.id} voided`);
  }
}
```

**NestJS Raw Body Configuration:**
```typescript
// In main.ts
app.useBodyParser('json', {
  verify: (req: any, res, buf) => {
    req.rawBody = buf; // Save raw body for Stripe webhook verification
  }
});
```

---

### 4.4 Admin Finance Section UI

**Complexity:** High
**Repository:** `client-main`

**New Files:**
```
client-main/app/pages/finance/ (NEW directory)
client-main/app/pages/finance/index.vue (NEW)
client-main/app/pages/finance/invoices.vue (NEW)
client-main/app/components/pages/finance/InvoiceTable.vue (NEW)
client-main/app/components/pages/finance/ManualChargeDialog.vue (NEW)
client-main/app/components/pages/finance/AddCreditDialog.vue (NEW)
client-main/app/gateway/billing/billing.gateway.ts (enhance)
```

**Implementation:**
```vue
<!-- finance/invoices.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { BillingGateway } from '~/gateway/billing'
import InvoiceTable from '~/components/pages/finance/InvoiceTable.vue'
import ManualChargeDialog from '~/components/pages/finance/ManualChargeDialog.vue'
import AddCreditDialog from '~/components/pages/finance/AddCreditDialog.vue'

definePageMeta({
  middleware: ['auth', 'admin'],
  layout: 'default'
})

const invoices = ref([])
const loading = ref(false)
const pagination = ref({ page: 0, perPage: 50, total: 0 })

const fetchInvoices = async () => {
  loading.value = true
  try {
    const response = await BillingGateway.getInvoices({
      page: pagination.value.page,
      perPage: pagination.value.perPage
    })
    invoices.value = response.list
    pagination.value.total = response.meta.total
  } finally {
    loading.value = false
  }
}

const handleVoid = async (invoiceId: string) => {
  if (!confirm('Are you sure you want to void this invoice?')) return
  await BillingGateway.voidInvoice({ id: invoiceId })
  await fetchInvoices()
}

const handleManualCharge = async (data: any) => {
  await BillingGateway.manualCharge(data)
  await fetchInvoices()
}

const handleAddCredit = async (data: any) => {
  await BillingGateway.addCredit(data)
  await fetchInvoices()
}

onMounted(fetchInvoices)
</script>

<template>
  <div class="p-6">
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-3xl font-bold">Invoices</h1>
      <div class="flex gap-2">
        <ManualChargeDialog @submit="handleManualCharge" />
        <AddCreditDialog @submit="handleAddCredit" />
      </div>
    </div>

    <InvoiceTable
      :invoices="invoices"
      :loading="loading"
      :pagination="pagination"
      @void="handleVoid"
      @page-change="fetchInvoices"
    />
  </div>
</template>

<!-- InvoiceTable.vue -->
<script setup lang="ts">
const props = defineProps<{
  invoices: any[]
  loading: boolean
  pagination: any
}>()

const emit = defineEmits<{
  void: [id: string]
  pageChange: [page: number]
}>()

const getStatusColor = (status: string) => {
  switch (status) {
    case 'Paid': return 'bg-green-100 text-green-800'
    case 'Open': return 'bg-yellow-100 text-yellow-800'
    case 'Failed': return 'bg-red-100 text-red-800'
    case 'Void': return 'bg-gray-100 text-gray-800'
    default: return 'bg-gray-100 text-gray-800'
  }
}
</script>

<template>
  <Table>
    <TableHeader>
      <TableRow>
        <TableHead>Partner</TableHead>
        <TableHead>Period</TableHead>
        <TableHead>Amount</TableHead>
        <TableHead>Status</TableHead>
        <TableHead>Payment Date</TableHead>
        <TableHead>Actions</TableHead>
      </TableRow>
    </TableHeader>
    <TableBody>
      <TableRow v-for="invoice in invoices" :key="invoice._id">
        <TableCell>{{ invoice.partner.name }}</TableCell>
        <TableCell>{{ invoice.billingPeriod }}</TableCell>
        <TableCell>${{ invoice.amount.toFixed(2) }}</TableCell>
        <TableCell>
          <Badge :class="getStatusColor(invoice.stripeStatus)">
            {{ invoice.stripeStatus }}
          </Badge>
        </TableCell>
        <TableCell>
          {{ invoice.paymentDate ? formatDate(invoice.paymentDate) : '-' }}
        </TableCell>
        <TableCell>
          <DropdownMenu>
            <DropdownMenuTrigger as-child>
              <Button variant="ghost" size="sm">
                <MoreHorizontal class="h-4 w-4" />
              </Button>
            </DropdownMenuTrigger>
            <DropdownMenuContent>
              <DropdownMenuItem>View Details</DropdownMenuItem>
              <DropdownMenuItem>Download PDF</DropdownMenuItem>
              <DropdownMenuItem
                v-if="invoice.stripeStatus === 'Open'"
                @click="emit('void', invoice._id)"
              >
                Void Invoice
              </DropdownMenuItem>
            </DropdownMenuContent>
          </DropdownMenu>
        </TableCell>
      </TableRow>
    </TableBody>
  </Table>
</template>
```

**Backend Endpoints:**
```typescript
// In BillingController
@Get('invoices')
@RouteGuard({ role: IUsers.Enum.Role.Admin })
async getInvoices(@Query() query: GetInvoicesDto) {
  return this.invoiceService.getMany(query);
}

@Post('manual-charge')
@RouteGuard({ role: IUsers.Enum.Role.Admin })
async manualCharge(@Body() dto: ManualChargeDto) {
  const partner = await this.partnersService.getOne({ _id: dto.partnerId });

  const charge = await this.stripeService.manualCharge(
    partner.stripeCustomerId,
    dto.amount * 100,
    dto.description
  );

  return charge;
}

@Post('add-credit')
@RouteGuard({ role: IUsers.Enum.Role.Admin })
async addCredit(@Body() dto: AddCreditDto) {
  const partner = await this.partnersService.getOne({ _id: dto.partnerId });

  const credit = await this.stripeService.addCredit(
    partner.stripeCustomerId,
    dto.amount * 100,
    dto.description
  );

  return credit;
}

@Patch('invoices/:id/void')
@RouteGuard({ role: IUsers.Enum.Role.Admin })
async voidInvoice(@Param('id', ParseMongoIdPipe) id: Types.ObjectId) {
  const invoice = await this.invoiceService.getOne({ _id: id });

  await this.stripeService.voidInvoice(invoice.stripeInvoiceId);

  await this.invoiceService.updateById(id, {
    stripeStatus: 'Void'
  });

  return invoice;
}
```

---

## SPRINT 5: Frontend Pages & UX (2 weeks)

### Goal
Complete user-facing features and UI polish.

### 5.1 Appointments Page

**Complexity:** Medium
**Repository:** Both

**Backend Files:**
```
api-main/src/modules/meetings/meetings.controller.ts
api-main/src/modules/meetings/dto/get-meetings.dto.ts (NEW)
```

**Frontend Files:**
```
client-main/app/pages/appointments/index.vue (NEW)
client-main/app/components/pages/appointments/AppointmentsTable.vue (NEW)
client-main/app/gateway/meetings/meetings.gateway.ts (NEW)
client-main/app/gateway/meetings/meetings.gateway.types.ts (NEW)
```

**Backend Endpoints:**
```typescript
// In MeetingsController
@Get()
@RouteGuard({ role: IUsers.Enum.Role.User })
@UseGuards(RoleGuard)
async getMany(
  @Query() query: GetMeetingsDto,
  @Req() req: InnerRequest
) {
  return this.meetingsService.getMany(query, req.user);
}

@Get(':id')
@RouteGuard({ role: IUsers.Enum.Role.User })
async getOne(@Param('id', ParseMongoIdPipe) id: Types.ObjectId) {
  return this.meetingsService.getOne({ _id: id });
}

// In MeetingsService
async getMany(query: GetMeetingsDto, user: User) {
  const filter: any = {};

  // Filter by partner if not owner
  if (user.role !== 'owner') {
    // Get leads for user's partner, then filter meetings
    const leads = await this.leadsRepository.getMany({
      filter: [{ column: 'partner', operator: 'eq', value: user.partner }]
    });
    filter.lead = { $in: leads.map(l => l._id) };
  }

  return this.repository.getManyPopulated({
    filter,
    limit: query.perPage,
    skip: query.page * query.perPage,
    sort: query.sort || [{ column: 'scheduledTime', direction: 'desc' }]
  });
}
```

**Frontend Implementation:**
```vue
<!-- appointments/index.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { MeetingsGateway } from '~/gateway/meetings'
import AppointmentsTable from '~/components/pages/appointments/AppointmentsTable.vue'

definePageMeta({
  middleware: ['auth'],
  layout: 'default'
})

const appointments = ref([])
const loading = ref(false)
const pagination = ref({ page: 0, perPage: 50, total: 0 })

const fetchAppointments = async () => {
  loading.value = true
  try {
    const response = await MeetingsGateway.getMany({
      page: pagination.value.page,
      perPage: pagination.value.perPage
    })
    appointments.value = response.list
    pagination.value.total = response.meta.total
  } finally {
    loading.value = false
  }
}

onMounted(fetchAppointments)
</script>

<template>
  <div class="p-6">
    <h1 class="text-3xl font-bold mb-6">Appointments</h1>

    <AppointmentsTable
      :appointments="appointments"
      :loading="loading"
      :pagination="pagination"
      @page-change="fetchAppointments"
    />
  </div>
</template>

<!-- AppointmentsTable.vue -->
<script setup lang="ts">
const props = defineProps<{
  appointments: any[]
  loading: boolean
  pagination: any
}>()

const getStatusBadge = (status: string) => {
  const colors = {
    Created: 'bg-blue-100 text-blue-800',
    Updated: 'bg-green-100 text-green-800',
    Rescheduled: 'bg-yellow-100 text-yellow-800',
    Canceled: 'bg-red-100 text-red-800'
  }
  return colors[status] || 'bg-gray-100 text-gray-800'
}
</script>

<template>
  <Table>
    <TableHeader>
      <TableRow>
        <TableHead>Lead</TableHead>
        <TableHead>Title</TableHead>
        <TableHead>Scheduled</TableHead>
        <TableHead>Status</TableHead>
        <TableHead>Assignee</TableHead>
        <TableHead>Host</TableHead>
        <TableHead>Actions</TableHead>
      </TableRow>
    </TableHeader>
    <TableBody>
      <TableRow v-for="apt in appointments" :key="apt._id">
        <TableCell>
          <NuxtLink :to="`/leads/${apt.lead._id}`" class="text-blue-600 hover:underline">
            {{ apt.leadEmail }}
          </NuxtLink>
        </TableCell>
        <TableCell>{{ apt.title || '-' }}</TableCell>
        <TableCell>{{ formatDateTime(apt.scheduledTime) }}</TableCell>
        <TableCell>
          <Badge :class="getStatusBadge(apt.status)">
            {{ apt.status }}
          </Badge>
        </TableCell>
        <TableCell>{{ apt.assigneeEmail || '-' }}</TableCell>
        <TableCell>{{ apt.hostEmail || '-' }}</TableCell>
        <TableCell>
          <Button variant="ghost" size="sm" @click="viewDetails(apt._id)">
            View
          </Button>
        </TableCell>
      </TableRow>
    </TableBody>
  </Table>
</template>
```

---

### 5.2 Lead Details - Appointments Section

**Complexity:** Medium
**Repository:** Both

**Backend:**
```typescript
// In LeadsService.getOne()
async getOnePopulated(filter: any) {
  const lead = await this.repository.findOne(filter)
    .populate('partner')
    .populate('billing');

  // Fetch meetings for this lead
  const meetings = await this.meetingsRepository.find({
    lead: lead._id
  }).sort({ scheduledTime: -1 });

  return {
    ...lead.toObject(),
    meetings
  };
}
```

**Frontend:**
```vue
<!-- LeadDetailsPage.vue - add to tabs -->
<Tabs default-value="overview">
  <TabsList>
    <TabsTrigger value="overview">Overview</TabsTrigger>
    <TabsTrigger value="appointments">Appointments</TabsTrigger>
    <TabsTrigger value="logs">Activity Logs</TabsTrigger>
  </TabsList>

  <TabsContent value="overview">
    <!-- Existing overview content -->
  </TabsContent>

  <TabsContent value="appointments">
    <AppointmentsSection :meetings="lead.meetings" />
  </TabsContent>

  <TabsContent value="logs">
    <!-- Activity logs (future) -->
  </TabsContent>
</Tabs>

<!-- AppointmentsSection.vue (NEW) -->
<script setup lang="ts">
const props = defineProps<{
  meetings: any[]
}>()
</script>

<template>
  <Card>
    <CardHeader>
      <CardTitle>Appointments</CardTitle>
    </CardHeader>
    <CardContent>
      <div v-if="meetings.length === 0" class="text-muted-foreground text-center py-8">
        No appointments scheduled
      </div>
      <div v-else class="space-y-4">
        <div
          v-for="meeting in meetings"
          :key="meeting._id"
          class="border rounded-lg p-4 hover:bg-accent/50 transition"
        >
          <div class="flex justify-between items-start">
            <div>
              <h4 class="font-semibold">{{ meeting.title || 'Meeting' }}</h4>
              <p class="text-sm text-muted-foreground">
                {{ formatDateTime(meeting.scheduledTime) }}
              </p>
            </div>
            <Badge :class="getStatusBadge(meeting.status)">
              {{ meeting.status }}
            </Badge>
          </div>
          <div class="mt-3 text-sm space-y-1">
            <div v-if="meeting.assigneeEmail">
              <span class="font-medium">Assignee:</span> {{ meeting.assigneeEmail }}
            </div>
            <div v-if="meeting.hostEmail">
              <span class="font-medium">Host:</span> {{ meeting.hostEmail }}
            </div>
            <div v-if="meeting.location">
              <span class="font-medium">Location:</span> {{ meeting.location }}
            </div>
          </div>
        </div>
      </div>
    </CardContent>
  </Card>
</template>
```

---

### 5.3 Advanced Filtering Enhancement

**Complexity:** Medium
**Repository:** `client-main`

**Files:**
```
client-main/app/components/sections/leads-table/LeadsDataTable.vue
client-main/app/components/ui/DateRangePicker.vue (use shadcn Calendar)
```

**Implementation:**
```vue
<!-- LeadsDataTable.vue - add to filter section -->
<script setup lang="ts">
import { ref, computed } from 'vue'
import { Calendar } from '~/components/shadcn-ui/calendar'
import { Popover, PopoverContent, PopoverTrigger } from '~/components/shadcn-ui/popover'
import { Select, SelectContent, SelectItem, SelectTrigger } from '~/components/shadcn-ui/select'

const filters = ref({
  dealStatus: [],
  scheduledDate: { from: null, to: null },
  salesManager: null,
  partner: null
})

const buildFilterQuery = () => {
  const filterArray = []

  if (filters.value.dealStatus.length) {
    filterArray.push({
      column: 'dealStatus',
      operator: 'in',
      value: filters.value.dealStatus
    })
  }

  if (filters.value.scheduledDate.from) {
    filterArray.push({
      column: 'scheduledDate',
      operator: 'gte',
      value: filters.value.scheduledDate.from
    })
  }

  if (filters.value.scheduledDate.to) {
    filterArray.push({
      column: 'scheduledDate',
      operator: 'lte',
      value: filters.value.scheduledDate.to
    })
  }

  if (filters.value.salesManager) {
    filterArray.push({
      column: 'sales_manager',
      operator: 'eq',
      value: filters.value.salesManager
    })
  }

  if (filters.value.partner) {
    filterArray.push({
      column: 'partner',
      operator: 'eq',
      value: filters.value.partner
    })
  }

  return filterArray
}

const applyFilters = async () => {
  tableState.filter = buildFilterQuery()
  await fetchLeads()
}

const clearFilters = () => {
  filters.value = {
    dealStatus: [],
    scheduledDate: { from: null, to: null },
    salesManager: null,
    partner: null
  }
  applyFilters()
}
</script>

<template>
  <div class="mb-4 flex flex-wrap gap-2">
    <!-- Deal Status Filter -->
    <Select v-model="filters.dealStatus" multiple>
      <SelectTrigger class="w-[180px]">
        Deal Status
      </SelectTrigger>
      <SelectContent>
        <SelectItem value="new">New</SelectItem>
        <SelectItem value="scheduled">Scheduled</SelectItem>
        <SelectItem value="in_touch">In Touch</SelectItem>
        <SelectItem value="won">Won</SelectItem>
        <SelectItem value="lost_no_show">Lost - No Show</SelectItem>
        <SelectItem value="lost_disqualified">Lost - Disqualified</SelectItem>
        <SelectItem value="lost_no_potential">Lost - No Potential</SelectItem>
      </SelectContent>
    </Select>

    <!-- Scheduled Date Range Filter -->
    <Popover>
      <PopoverTrigger as-child>
        <Button variant="outline" class="w-[280px] justify-start text-left">
          <CalendarIcon class="mr-2 h-4 w-4" />
          {{ filters.scheduledDate.from
            ? `${formatDate(filters.scheduledDate.from)} - ${formatDate(filters.scheduledDate.to || filters.scheduledDate.from)}`
            : 'Scheduled Date'
          }}
        </Button>
      </PopoverTrigger>
      <PopoverContent class="w-auto p-0" align="start">
        <Calendar
          v-model:from="filters.scheduledDate.from"
          v-model:to="filters.scheduledDate.to"
          mode="range"
          :number-of-months="2"
        />
      </PopoverContent>
    </Popover>

    <!-- Sales Manager Filter -->
    <Select v-model="filters.salesManager">
      <SelectTrigger class="w-[180px]">
        Sales Manager
      </SelectTrigger>
      <SelectContent>
        <SelectItem v-for="manager in salesManagers" :value="manager">
          {{ manager }}
        </SelectItem>
      </SelectContent>
    </Select>

    <!-- Partner Filter (Admin Only) -->
    <Select v-if="userGetters.isAdmin" v-model="filters.partner">
      <SelectTrigger class="w-[180px]">
        Partner
      </SelectTrigger>
      <SelectContent>
        <SelectItem v-for="partner in partners" :value="partner._id">
          {{ partner.name }}
        </SelectItem>
      </SelectContent>
    </Select>

    <Button @click="applyFilters">Apply</Button>
    <Button @click="clearFilters" variant="outline">Clear</Button>
  </div>
</template>
```

---

### 5.4 UI/UX Polish

**Complexity:** Low-Medium
**Repository:** `client-main`

**Tasks:**

1. **User Management Table**
   - Add sorting: Name (A→Z), Email, Role, Status, Created Date
   - Add pagination info: "Page X of Y"
   - Add "Assign Partner on Create" dropdown in create user dialog

2. **Leads Table**
   - Add separate email search bar (instant search)
   - Set default sorting: Deal Status (New → Old)
   - Add deal status dropdown in lead details "Overview" section

3. **Partner Settings**
   - Restrict access to slug and Meeting type for Users (admin-only fields)

4. **General UX**
   - Add loading skeletons (use shadcn Skeleton component)
   - Add empty states with helpful messages
   - Add toast notifications for actions (vue-sonner)
   - Consistent hover states on table rows
   - Keyboard shortcuts (Cmd+K for search)

**Implementation:**
```vue
<!-- Example: Add toast notifications -->
<script setup lang="ts">
import { toast } from 'vue-sonner'

const handleSave = async () => {
  try {
    await save()
    toast.success('Saved successfully!')
  } catch (error) {
    toast.error('Failed to save')
  }
}
</script>
```

---

## SPRINT 6: Bug Fixes & Technical Debt (1 week)

### Goal
Stabilize platform and address technical debt.

### 6.1 File Import Bug Fix

**Complexity:** Unknown (requires investigation)
**Repository:** `api-main`

**Files:**
```
api-main/src/modules/leads/leads.service.ts
api-main/src/modules/leads/dto/import.dto.ts
```

**Investigation Steps:**
1. Review import validation logic in `LeadsService`
2. Test with various CSV formats (encoding, delimiters, headers)
3. Check error handling for malformed files
4. Add logging for debugging

**Potential Issues:**
- CSV parsing failures (encoding issues)
- Missing validation for required fields
- Race conditions in bulk operations
- Memory issues with large files

**Fix Approach:**
- Add file size limits
- Validate file before processing
- Better error messages
- Add retry logic for failed rows

---

### 6.2 Token Security Improvements

**Complexity:** Medium
**Repository:** `api-main`

**Files:**
```
api-main/src/modules/auth/auth.service.ts
api-main/src/modules/partners/partners.service.ts
```

**Enhancements:**
1. **Token Rotation**: Generate new refresh token on each refresh
2. **Rate Limiting**: Add rate limits on `/auth/sign-in` and `/auth/token/refresh`
3. **Token Revocation List**: Use Redis for blacklisted tokens (more efficient)
4. **Secure Generation**: Already using `crypto.randomBytes` (good)

**Implementation:**
```typescript
// Token rotation
async refreshTokens(oldRefreshToken: string) {
  const decoded = this.jwtService.verify(oldRefreshToken);

  // Blacklist old refresh token
  await this.tokenRepository.blacklist(oldRefreshToken);

  // Generate new pair
  const { accessToken, refreshToken } = this.generateTokens(decoded.userId, decoded.role);

  return { accessToken, refreshToken };
}
```

---

## VERIFICATION & TESTING

### Phase 1 Verification
```bash
# Test scheduled_date sync
1. Book meeting via ChiliPiper webhook
2. Check lead's scheduledDate field updated
3. Verify index exists: db.leads.getIndexes()
4. Test with multiple meetings (latest should win)
```

### Phase 2 Verification
```bash
# Test email service
1. Request password reset → Check email received
2. Wait for 9 AM PT next day → Check reminder emails sent
3. Change lead to "lost" → Check julia@belkins.io received alert
4. Click magic link → Verify filters applied
```

### Phase 3 Verification
```bash
# Test user management
1. Toggle user inactive → Verify login blocked
2. Edit partner ICP → Verify saves correctly
3. Check admin sees partner column in leads
4. Users should NOT see partner column
```

### Phase 4 Verification
```bash
# Test Stripe integration
1. Wait for 1st of month → Check invoices generated
2. Send test Stripe webhook → Verify status updates
3. Admin manually charges partner → Check Stripe dashboard
4. Admin voids invoice → Check status in DB + Stripe
```

### Phase 5 Verification
```bash
# Test frontend features
1. Visit /appointments → Verify meetings displayed
2. Open lead details → Check appointments tab
3. Apply filters (date range, manager) → Verify results
4. Test on mobile → Check responsive layout
```

---

## DEPENDENCIES & PREREQUISITES

### External Services Setup

**1. Mailgun**
- Create account at mailgun.com
- Verify domain (mg.belkins.io)
- Setup DNS records (SPF, DKIM, MX)
- Get API key from dashboard
- Add to `.env`:
  ```env
  MAILGUN_DOMAIN=mg.belkins.io
  MAILGUN_API_KEY=key-xxx
  ```

**2. Stripe**
- Create account at stripe.com
- Get test API keys from dashboard
- Setup webhook endpoint: `https://api.partners.belkins.io/v1/billing/webhooks/stripe`
- Select events: `invoice.paid`, `invoice.payment_failed`, `invoice.voided`
- Get webhook secret
- Add to `.env`:
  ```env
  STRIPE_SECRET_KEY=sk_test_xxx
  STRIPE_WEBHOOK_SECRET=whsec_xxx
  ```
- Create customer records for existing partners (manual or migration script)

**3. MongoDB**
- Backup existing database: `mongodump --uri="old-connection"`
- Create new dedicated instance on MongoDB Atlas
- Import data: `mongorestore --uri="new-connection"`
- Update `.env` with new connection string

**4. Cron Monitoring**
- Setup Sentry Cron Monitoring (optional but recommended)
- Configure dead letter queue for failed jobs
- Add alerting for cron failures

---

## CRITICAL FILES SUMMARY

### API (Backend) - Critical Files

| File | Purpose | Sprint |
|------|---------|--------|
| `api-main/src/modules/leads/schemas/lead.schema.ts` | Add `scheduledDate` field | 1 |
| `api-main/src/modules/meetings/meetings.service.ts` | Real-time meeting→lead sync | 1 |
| `api-main/src/modules/users/schemas/users.schema.ts` | Add `isActive`, `resetToken` fields | 2-3 |
| `api-main/src/common/mailgun/mailgun.service.ts` | Email service wrapper | 2 |
| `api-main/src/modules/notifications/notifications.cron.ts` | Daily reminder cron | 2 |
| `api-main/src/modules/auth/auth.service.ts` | Password reset flow | 2 |
| `api-main/src/common/stripe/stripe.service.ts` | Stripe integration | 4 |
| `api-main/src/modules/billing/billing.cron.ts` | Monthly invoice generation | 4 |
| `api-main/src/modules/billing/stripe-webhook.processor.ts` | Stripe webhook handling | 4 |
| `api-main/src/app.module.ts` | Import ScheduleModule | 2 |

### Client (Frontend) - Critical Files

| File | Purpose | Sprint |
|------|---------|--------|
| `client-main/app/pages/forgot-password.vue` | Password reset request | 2 |
| `client-main/app/pages/reset-password.vue` | Password reset form | 2 |
| `client-main/app/pages/users.vue` | User active/inactive toggle | 3 |
| `client-main/app/pages/partners/[slug]/settings.vue` | Partner ICP editor | 3 |
| `client-main/app/pages/finance/invoices.vue` | Admin finance UI | 4 |
| `client-main/app/pages/appointments/index.vue` | Appointments page | 5 |
| `client-main/app/components/sections/leads-table/LeadsDataTable.vue` | Advanced filtering | 5 |
| `client-main/app/gateway/billing/billing.gateway.ts` | Finance API calls | 4 |
| `client-main/app/gateway/meetings/meetings.gateway.ts` | Meetings API calls | 5 |

---

## RISK MITIGATION

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stripe webhook failures** | Invoices not updated | Idempotency keys, event logging, manual reconciliation tool |
| **Cron job silent failures** | No emails sent | Dead letter queue, failure alerts to admin, manual trigger endpoint |
| **Database migration data loss** | Business disruption | Full backup before migration, test in staging, rollback plan |
| **Email deliverability** | Users don't receive emails | SPF/DKIM setup, test emails, bounce handling, fallback notifications |
| **ChiliPiper webhook downtime** | Meetings not synced | Queue retry logic, manual sync button, backfill script |

---

## DEPLOYMENT STRATEGY

### Staging Rollout
1. Deploy Phase 1 → Test 3 days → Production
2. Deploy Phase 2 with cron disabled → Test manually → Enable cron
3. Deploy Phase 3 → Direct to production (low risk)
4. Deploy Phase 4 with webhook handler → Test in Stripe dashboard → Enable auto-billing
5. Deploy Phase 5 → Gradual rollout with feature flags

### Rollback Plan
- Keep previous API Docker image running
- Feature flags for new UI components
- Database backups before each deployment
- Ability to disable cron jobs via environment variable

---

## SUCCESS METRICS

### Sprint 1
- ✅ 100% of meetings sync `scheduledDate` to leads
- ✅ Zero sync failures in last 7 days

### Sprint 2
- ✅ Daily emails sent at 9 AM PT (verify logs)
- ✅ <1% email bounce rate
- ✅ Lost lead alerts arrive within 1 minute

### Sprint 3
- ✅ Admins can toggle user status
- ✅ Partner ICP saves successfully
- ✅ Partner column visible to admins only

### Sprint 4
- ✅ Invoices auto-generated on 1st of month
- ✅ 100% webhook processing success
- ✅ Admin manual controls functional

### Sprint 5
- ✅ Appointments page loads <2s
- ✅ Lead details shows all appointments
- ✅ All filters work correctly

---

This comprehensive plan provides clear implementation guidance for all features across 6 sprints. Each phase includes detailed code examples, database considerations, testing strategies, and risk mitigation approaches.
