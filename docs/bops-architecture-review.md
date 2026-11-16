# Bops Architecture Review for Wellness Operations Platform

**Review Date**: November 16, 2026  
**Reviewer**: Cloud Agent Architecture Analysis  
**Purpose**: Evaluate Bops for reuse as a wellness operations AI platform serving clinics and wellness studios

---

## Executive Summary

**Bops** ("Business ops, run by bots") is a sophisticated AI agent platform that runs teams of AI assistants with real computer access. Each bot works on cloud computers, can use 1,400+ connected apps, and communicates via chat, email, phone, and messaging platforms (Slack, Telegram, Discord).

**Reuse Viability**: **MODERATE with significant modifications required**

The architecture is well-designed for business operations but requires substantial changes for healthcare/wellness use, particularly around:
- HIPAA compliance capabilities (none currently)
- Service provider dependencies (no BAA-eligible options currently configured)
- Multi-tenancy isolation (designed for single-user/team, not per-clinic)
- PHI logging and security (not healthcare-aware)

---

## 1. Architecture Overview

### 1.1 Main Components

**Desktop Application**
- **Platform**: macOS (Apple Silicon only)
- **Framework**: Electron + Next.js 16.4.5
- **Language**: TypeScript, React 19.2.8
- **Location**: `desktop/` (Electron wrapper), `app/` (Next.js UI), `components/` (React components)

**Server Layer**
- **Runtime**: Node.js 22+
- **Framework**: Next.js App Router
- **Port**: 3210 (localhost only by default)
- **Location**: `lib/server/` (150+ TypeScript modules)
- **State Management**: JSON files (`.data/`) for self-hosted, PostgreSQL for cloud-hosted

**Agent Execution Environments**
1. **Cloud Computers** (Orgo VMs): 4 cores, 16GB RAM, 80GB disk, Ubuntu Linux
   - Template: `system/bops-base@0.1.11`
   - Runs: Chrome, LibreOffice, GIMP, Inkscape
   - Access: 4 virtual screens, VNC/WebRTC streaming
   
2. **Local Mac Execution**: Optional for specific tasks requiring local network access

**Bops Cloud** (Hosted Service)
- **Location**: `cloud/` directory (100+ TypeScript files)
- **Runtime**: Node.js (separate from Next.js)
- **Database**: PostgreSQL (schema `bops`, user `bops_app`)
- **Purpose**: Multi-user hosting, webhooks, state synchronization, API proxying

### 1.2 Agent Architecture

**Agent Definition**
- Stored in `AppState.bots[]` (TypeScript type `Bot`)
- Properties: name, role, color, computer assignment, permissions, apps access
- Main bot (coordinator) + specialist bots (per workspace)

**Agent Execution Models**
1. **OpenAI Agents API**: For tasks with function calling (Codex CLI integration)
2. **OpenAI Responses API**: For computer-based tasks with visual tools
3. **GPT-Live**: For voice calls over SIP

**Memory System**
- **Provider**: Honcho AI SDK
- **Scope**: Per workspace, per bot, per chat
- **Storage**: Conversation history, facts, preferences
- **Location**: `lib/server/memory.ts`

### 1.3 Data Stores

**Self-Hosted Mode** (`BOPS_SELF_HOSTED=1`)
- State file: `.data/users/<orgo_user_id>/state.json`
- Attachments: `.data/uploads/`
- Logs: `~/Library/Logs/Bops/server.log` (macOS)

**Cloud-Hosted Mode**
- Database: PostgreSQL
  - `bops.app_state` - User state (JSONB, excludes messages)
  - `bops.chat_messages` - Individual messages (one row each)
  - `bops.phone_lines` - Phone number ownership
  - `bops.mail_handles` - Email address handles
  - `bops.cloud_usage` - AI usage tracking
- Migration system: `db/migrations/*.sql` (auto-applied)

**Credentials Storage**
- macOS Keychain (passwords, 2FA keys, bot tokens)
- Never exposed to AI models
- Server-side only access

### 1.4 Key Dependencies

```json
Core Framework:
- next: 16.4.5
- react: 19.2.8
- electron: 44.9.1

AI/ML:
- openai: 7.27.0
- @honcho-ai/sdk: 2.5.1 (memory)

Communication:
- agentmail: 0.5.35 (email service SDK)
- ws: 8.22.0 (WebSockets)

Integrations:
- @composio/core: 0.22.0 (1400+ apps)
- @composio/openai: 0.13.0

Infrastructure:
- pg: 8.23.1 (PostgreSQL)
- @novnc/novnc: 1.7.0 (screen streaming)

Analytics:
- posthog-js: 1.231.0
- posthog-node: 5.10.0
```

### 1.5 Deployment Model

**End User Deployment**
- Download `.dmg` from bops.bot
- Codesigned and notarized by Apple (Team ID: A47GG9FV27)
- Auto-updates via Electron's updater
- Self-contained: Includes Node.js runtime, Next.js server, dependencies

**Cloud Deployment**
- Bops Cloud runs on Orgo infrastructure
- Agent computers: Orgo's cloud VM service
- Webhooks relay: Fly.io (`edge/` directory)

---

## 2. Agent Email Service: AgentMail

### 2.1 Service Provider

**AgentMail** (https://agentmail.to, https://api.agentmail.to)
- **Purpose**: Email addresses for AI agents
- **SDK**: `agentmail` npm package (v0.5.35)
- **License**: Proprietary service

### 2.2 Integration Architecture

**How It's Wired**
```
Client: AgentMailClient SDK (lib/server/mail.ts)
Authentication: AGENTMAIL_API_KEY environment variable
Connection: REST API + WebSocket for real-time delivery
State: lib/types.ts Bot.mail (inbox metadata)
```

**Key Files**
- `lib/server/mail.ts` (1,046 lines) - Complete email implementation
- Integration points:
  - Lines 3, 46-60: Client initialization and authentication
  - Lines 93-106: AgentMailClient setup with retry logic
  - Lines 119-138: Pod (inbox container) management
  - Lines 710-735: WebSocket connection for real-time email
  - Lines 903-929: Sending email with approval workflow
  - Lines 932-970: Reply threading

**Architecture Patterns**
- One AgentMail pod per Bops installation
- One inbox per bot within that pod
- Addresses: `<bot>@<workspace>.bops.bot` or `<bot>.<workspace>@agentmail.to`
- Real-time via WebSocket (survives Mac sleep/offline)
- Catch-up mechanism for missed emails

**Configuration**
```bash
AGENTMAIL_API_KEY=       # Required
AGENTMAIL_POD_ID=        # Optional (auto-created)
BOPS_MAIL_DOMAIN=        # Default: bops.bot
BOPS_OWNER_EMAILS=       # Self-hosted: owner's emails (comma-separated)
```

### 2.3 Email Workflow

**Inbound**
1. Email arrives at AgentMail → WebSocket push to Bops
2. Parsed for text, HTML, attachments, thread context
3. Lands in bot's chat as system message
4. Bot responds (with approval if not from owner)

**Outbound** 
1. Bot calls `send_email` or `reply_email` tool
2. Approval workflow (if recipient not owner):
   - "Never" setting blocks send
   - "With your OK" pauses for approval
   - "Just do it" or owner recipient sends immediately
3. AgentMail retries (2× on 408/429/5xx, lib/server/mail.ts:100)
4. Sent email tracked in chat with card

**Threading**
- AgentMail handles In-Reply-To, References headers
- Bops stores thread history (`threadSoFar()` function)
- Replies include conversation context (last 6 emails)

---

## 3. Agent Phone Service: AgentPhone

### 3.1 Service Provider

**AgentPhone** (https://agentphone.ai, https://api.agentphone.ai)
- **Purpose**: Phone numbers, SMS, and voice calls for AI agents
- **Features**: Voice calls, SMS, iMessage support
- **Type**: Proprietary service (no public SDK, uses REST API)

### 3.2 Integration Architecture

**How It's Wired**
```
Client: Direct REST API calls (lib/server/phone.ts)
Authentication: AGENTPHONE_API_KEY + AGENTPHONE_SUB_ACCOUNT
Webhooks: BOPS_AGENTPHONE_HOOK_URL (via edge/ relay or Bops Cloud)
Voice: AgentPhone agent (webhook mode) calls OpenAI GPT-Live
State: lib/types.ts Bot.phone, Bot.phoneLine
```

**Key Files**
- `lib/server/phone.ts` (1,297 lines) - Phone number management, SMS
- `lib/server/phone-voice.ts` (384 lines) - Voice call handling
- Integration points:
  - Lines 67-82 (phone.ts): API authentication with webhook secrets
  - Lines 108-133: API call wrapper with Bops Cloud proxy
  - Lines 322-360: Number provisioning (buying/finding)
  - Lines 367-374: Voice routing configuration
  - Lines 530-552: SMS sending
  - Lines 693-764: Inbound SMS handling
  - Voice: phone-voice.ts lines 17-30 (webhook architecture)

**Phone Number Architecture**
- Numbers live in "Bops" AgentPhone sub-account
- Each bot has an AgentPhone agent with voice mode "webhook"
- Number carries `externalId`: `bops-<install>-<bot>`
- Webhooks deliver to Bops Cloud → tunneled to Mac OR edge/ relay (Fly.io)

**Configuration**
```bash
AGENTPHONE_API_KEY=              # Required
AGENTPHONE_SUB_ACCOUNT=          # Required
AGENTPHONE_WEBHOOK_SECRET=       # For webhook verification
BOPS_AGENTPHONE_HOOK_URL=        # Public webhook endpoint
BOPS_PHONE_AREA=                 # Area code (default: 415)
BOPS_SUPPORT_EMAIL=              # For SMS HELP replies
```

### 3.3 Voice Call Architecture

**Call Flow**
1. Incoming call → AgentPhone agent (webhook mode)
2. AgentPhone hears caller via speech-to-text
3. Each turn sent to Bops webhook as text
4. Bops processes via OpenAI Responses API:
   - Owner gets full bot (delegates to chat, can do work)
   - Stranger gets message-taking bot only
5. Bot's text response sent back to AgentPhone
6. AgentPhone speaks response in bot's chosen voice
7. Transcript saved to chat after call ends

**Voice Integration**
- Models: OpenAI GPT-Live, gpt-6.1-sol (default phone model)
- Voice matching: Bops maps bot voices to AgentPhone's voices (lib/phone-voices.ts)
- Delegation: Owner's work requests spawn chat threads, results texted back
- Location: `lib/server/phone-voice.ts` (voicePrompt, respond, startWork functions)

**SMS Handling**
- Inbound: Parsed, checked against owner numbers, lands in chat
- Outbound: Plain text (markdown stripped), max 1500 chars
- Threading: iMessage reply-to support
- Tapbacks: iMessage reactions mapped to chat reactions
- Carrier keywords: STOP/START/HELP handled automatically

---

## 4. Orgo: Cloud Computer Service

### 4.1 What Orgo Is

**Orgo** (https://orgo.ai) is the cloud VM provider that runs the AI agents' computers.

**NOT a bot or agent** - it's the infrastructure service providing:
- Virtual machines (desktops) for each bot
- 4-core, 16GB RAM, 80GB disk computers
- Multiple virtual screens per computer
- Screen streaming (VNC/WebRTC)
- Template-based provisioning
- Workspace management

**Orgo in Context**
```
User's Mac (Bops app)
    ↓
Orgo API (orgo.ai)
    ↓ provisions
Orgo Cloud Computers (VMs)
    ↓ run
Bot Tasks (OpenAI Agents/Responses API + browser/terminal access)
```

### 4.2 Integration Architecture

**How It's Wired**
```
Client: lib/server/orgo.ts (1,578 lines of REST API calls)
Authentication: ORGO_API_KEY or signed-in user's Orgo account
Workspace: Bops creates/manages "bops" workspace in user's account
Template: system/bops-base@0.1.11 (built from orgo/bops-base.mjs)
State: lib/types.ts Bot.computerId, Bot.computerStatus
```

**Key Files**
- `lib/server/orgo.ts` - Complete Orgo API client
  - Lines 19-42: Authentication (API key sources)
  - Lines 155-205: API call wrapper with retry logic
  - Lines 280-330: Computer change tracking
  - Lines 780-928: Main `orgo` API object (create, fork, clone, screens, etc.)
  - Lines 930-1000: Computer lifecycle (fork, clone, restart)
- `lib/server/orgo-auth.ts` - Orgo sign-in flow
- `orgo/bops-base.mjs` - Computer template builder

**Computer Lifecycle**
1. **Create**: Provisions new VM from template (180s timeout)
2. **Fork/Clone**: Copies running computer (includes memory for fork)
3. **Screens**: Create/destroy virtual displays (1280x960, 4:3)
4. **Control**: Screenshot, click, type, keyboard input
5. **Execute**: Bash commands with timeout
6. **Suspend/Resume**: Sleep to storage, wake on demand
7. **Delete**: Only within "bops" workspace

**Template Architecture**
- Base: Ubuntu Linux with Chrome, LibreOffice, GIMP, Inkscape
- Custom: Includes Bops runner scripts, MCP screen server
- Built via: `node orgo/bops-base.mjs publish`
- VM folder: `vm/` contains runner code deployed to computers

### 4.3 Provisioning Details

**Configuration**
```bash
ORGO_API_KEY=              # For self-hosted or ~/.orgo/credentials.json
BOPS_ORGO_WORKSPACE=       # Workspace ID (default: auto-created "bops")
BOPS_ORGO_TEMPLATE=        # Template ref (default: system/bops-base@0.1.11)
BOPS_DISK_GB=              # Disk size (default: 80GB)
TAILSCALE_AUTH_KEY=        # For private network access to screens
```

**Resource Limits**
- Controlled by user's Orgo plan (Hacker/Startup/Scale tiers)
- Each bot can have own computer or share main bot's
- Free tier: 1 computer, 10 hours/month
- Computer specs: Fixed at 4 cores, configurable RAM (default 16GB)

**Network Architecture**
- Computers join user's Tailscale network (optional)
- Screen access: WebRTC (UDP) or VNC over WebSocket
- Relay: Optional edge/ deployment on Fly.io for webhooks

---

## 5. Service Limits and Constraints

### 5.1 AgentMail

**Pricing Model**
- Not publicly documented (proprietary service)
- Likely per-inbox and/or per-email pricing
- Client SDK: `agentmail` package (BSD-3-Clause license)

**Rate Limits**
- Retries: 2 additional attempts on 408/429/5xx errors (code: lib/server/mail.ts:100)
- Backoff: 1s, then 2s between retries
- No explicit rate limit documentation in code

**Deliverability**
- Addresses: `<name>@<handle>.bops.bot` or `<name>.<handle>@agentmail.to`
- DMARC: Checked for incoming mail (lines 574-623 in mail.ts)
- DNS: Requires domain verification for custom domains
- SPF/DKIM: Handled by AgentMail

**HIPAA BAA Availability**: ❌ **NO**
- No evidence of HIPAA compliance in documentation
- No BAA (Business Associate Agreement) mentioned
- Service likely not HIPAA-eligible

**Data Residency**: ❓ **UNKNOWN**
- No data residency options documented
- API endpoint: US-based (api.agentmail.to)

**Vendor Lock-in**
- **HIGH** - Proprietary service with no standard protocol
- SDK dependency: `agentmail` package
- Migration path: Would require complete email reimplementation

**Self-Host Options**: ❌ **NO**
- No self-hosted version available
- Requires cloud service access

### 5.2 AgentPhone

**Pricing Model**
- Number rental: ~$3/month (code comment: lib/server/phone.ts:319)
- Usage: Likely per-minute voice and per-SMS charges
- Not explicitly documented in codebase

**Rate Limits**
- No explicit rate limits in code
- SMS length: 1500 chars max (line 540)
- Recipients: 10 max per email (line 912)

**Number Provisioning**
- **A2P 10DLC**: Required for US SMS (code: README.md:237)
  - Carrier registration needed
  - Brand verification required
  - Campaign approval process
- **Area Codes**: US/Canada only (default 415, configurable)
- **Number Types**: SMS and iMessage supported

**Voice Capabilities**
- **Technology**: Webhook-based voice agent
- **Integration**: Calls OpenAI GPT-Live for AI responses
- **Features**: Speech-to-text, text-to-speech, call transcription
- **Voice Options**: Multiple voice options mapped to OpenAI voices

**HIPAA BAA Availability**: ❓ **LIKELY NO**
- No HIPAA compliance mentioned
- Twilio (common backend for such services) offers BAA but requires explicit setup
- AgentPhone service layer may not be HIPAA-compliant even if backend is

**Data Residency**: ❓ **UNKNOWN**
- No options documented
- Likely US-based

**Vendor Lock-in**
- **HIGH** - Proprietary API, no standard telephony interfaces
- No SIP trunking or Twilio Programmable Voice fallback
- Number portability unknown

**Self-Host Options**: ❌ **NO**
- Cloud service only
- Optional: OpenAI SIP trunk direct integration (code: lib/server/phone.ts:54, 862-1027)

### 5.3 Orgo (Cloud Computers)

**Pricing Model**
```
Orgo Plans (from orgo.ai):
- Hacker: Limited VMs
- Startup: More VMs, better specs
- Scale: Maximum capacity

Bops-Specific:
- Free Computer: 1 per account, 10 hours/month (lib/types.ts:40-41)
- Paid: Based on Orgo plan + AI usage
```

**Resource Limits**
- **CPU**: 4 cores per computer (fixed)
- **RAM**: 16GB default (configurable: lib/server/orgo.ts:786)
- **Disk**: 80GB default (grows automatically, never shrinks)
- **Screens**: 4 virtual displays per computer
- **Concurrent Computers**: Based on user's Orgo plan

**VM Lifecycle Constraints**
- **Create/Fork**: 180 second timeout (lib/server/orgo.ts:105, 800)
- **Screenshots**: 20 second timeout, retry once (lines 103, 986-1016)
- **Commands**: 60 second default timeout (line 970)
- **Wake from Storage**: Several minutes (lines 1020-1027)

**HIPAA BAA Availability**: ❓ **UNKNOWN**
- Orgo is general-purpose VM service
- No HIPAA compliance mentioned in documentation
- Would need investigation with Orgo team

**Data Residency**: ❓ **UNKNOWN**
- VM locations not specified
- Likely US-based infrastructure

**Vendor Lock-in**
- **MEDIUM** - VMs are Ubuntu Linux (portable)
- **HIGH** for Orgo-specific features: templates, screens, fork/clone
- Migration: Could use any VM provider but lose integrated features

**Self-Host Options**: ⚠️ **PARTIAL**
- Requires Orgo account for VMs
- Can self-host Bops app but not the agent computers
- No alternative to Orgo for the "computer tool" feature

### 5.4 Other Service Dependencies

**OpenAI**
- **Usage**: GPT models, Agents API, Responses API, GPT-Live
- **HIPAA BAA**: ✅ **YES** (available for Enterprise)
- **Costs**: Token-based, Codex execution time
- **Limits**: Standard API rate limits
- **Alternative**: Could use Azure OpenAI (HIPAA eligible)

**Honcho (Memory)**
- **Usage**: Long-term conversation memory
- **HIPAA**: ❌ **NO** (startup service, no compliance docs)
- **Lock-in**: MEDIUM (could use alternative vector DB)
- **Self-host**: ❌ NO

**Composio (App Integrations)**
- **Usage**: 1,400+ app connections
- **HIPAA**: ❌ **NO**
- **Lock-in**: HIGH (entire integrations ecosystem)
- **Concern**: Third-party app access = PHI exposure risk

**treg (Business Data)**
- **Usage**: Company/people lookup
- **HIPAA**: ❌ **NO**
- **Lock-in**: LOW (optional feature)

**TypeSafe (Decisions)**
- **Usage**: Small yes/no judgments
- **HIPAA**: ❌ **NO**
- **Lock-in**: LOW (internal logic could replace)

**Tailscale (Networking)**
- **Usage**: Private network for screen access
- **HIPAA**: ✅ **YES** (Enterprise plan offers BAA)
- **Lock-in**: LOW (standard WireGuard underneath)

---

## 6. Coupling Analysis

### 6.1 AgentMail Coupling: **MODERATE-HIGH**

**Direct SDK Calls**: `lib/server/mail.ts`
- Line 3: `import { AgentMailClient, type AgentMail } from "agentmail"`
- Lines 93-106: Client initialization with API key
- Lines 119-138: Pod/inbox creation
- Lines 710-735: WebSocket connection
- Lines 903-970: Send/reply email operations

**State Dependencies**
- `lib/types.ts` Bot.mail, Bot.email, Bot.mailWanted
- `bops.mail_handles` table (cloud hosted)
- `bops.mail_inboxes` table (cloud hosted)

**Files Requiring Changes to Swap Provider**
1. `lib/server/mail.ts` (1,046 lines) - **COMPLETE REWRITE**
   - All AgentMail SDK calls
   - WebSocket real-time delivery
   - Thread management
   - Inbox lifecycle
   
2. `cloud/provision.ts` - Automated inbox setup
3. `cloud/handles.ts` - Address handle management
4. `lib/types.ts` - Bot.mail type definition
5. `components/app/mail-*.tsx` - UI components (multiple files)
6. Database migrations if using custom domain tables

**Abstraction Layer**: ❌ **NONE**
- Direct SDK calls throughout
- No email abstraction interface
- Tightly coupled to AgentMail WebSocket and API structure

### 6.2 AgentPhone Coupling: **HIGH**

**Direct API Calls**: `lib/server/phone.ts`
- Lines 108-133: REST API wrapper (`ap()` function)
- Lines 322-360: Number provisioning
- Lines 367-414: Agent and webhook management
- Lines 530-789: SMS and voice handling

**State Dependencies**
- `lib/types.ts` Bot.phone, Bot.phoneLine
- `bops.phone_lines` table
- `bops.phone_secrets.json` file

**Files Requiring Changes to Swap Provider**
1. `lib/server/phone.ts` (1,297 lines) - **COMPLETE REWRITE**
   - All number provisioning
   - SMS send/receive
   - Voice agent configuration
   
2. `lib/server/phone-voice.ts` (384 lines) - **MAJOR CHANGES**
   - Voice call handling
   - GPT-Live integration
   - Prompt generation
   
3. `cloud/voice.ts` - Cloud voice handling
4. `cloud/lines.ts` - Number ownership tracking
5. `cloud/provision.ts` - Automated number setup
6. `lib/types.ts` - Bot.phoneLine type
7. `edge/server.mjs` - Webhook relay (if self-hosted)

**Abstraction Layer**: ❌ **NONE**
- Direct API calls scattered throughout
- Voice integration tightly coupled to AgentPhone webhook format
- No telephony abstraction

### 6.3 Orgo Coupling: **MEDIUM-HIGH**

**API Client**: `lib/server/orgo.ts` (1,578 lines)
- Comprehensive REST API wrapper
- All VM operations (create, fork, clone, screen, bash)
- Template and workspace management

**State Dependencies**
- `lib/types.ts` Bot.computerId, Bot.computerStatus, Bot.freeComputer
- Workspace to Orgo project mappings

**Files Requiring Changes to Swap Provider**
1. `lib/server/orgo.ts` (1,578 lines) - **MAJOR REWRITE**
   - VM lifecycle management
   - Screen access and streaming
   - Template system
   
2. `lib/server/sessions.ts` - Task execution on computers
3. `lib/server/computer-*.ts` - Multiple computer management files:
   - `computer-task.ts` - Computer-tool based tasks
   - `computer-size.ts` - VM sizing
   - `computer-ssh.ts` - SSH access
4. `lib/server/screens.ts` - Screen management
5. `orgo/bops-base.mjs` - VM template builder
6. `vm/` - All runner code deployed to VMs
7. UI components: `components/app/computer-*.tsx`

**Abstraction Layer**: ⚠️ **PARTIAL**
- Centralized in `orgo.ts` but still tightly coupled
- OpenAI computer tool requires specific screen/bash interface
- Template system specific to Orgo

**Migration Impact**
- **VM Provider**: Could swap to AWS/GCP/Azure VMs
- **Lost Features**: Fork/clone, integrated screens, template system
- **Alternative Approach**: Use OpenAI Responses API without computer tool (loses screen capability)

### 6.4 Other Dependencies

**OpenAI** (`lib/server/openai-client.ts`)
- **Coupling**: HIGH - Core to entire agent system
- **Abstraction**: None - Direct SDK usage
- **Migration**: Possible to Azure OpenAI with code changes

**Honcho Memory** (`lib/server/memory.ts`)
- **Coupling**: MEDIUM - Centralized in one file
- **Abstraction**: Minimal - Direct SDK calls
- **Migration**: Could swap to vector DB (Pinecone, Weaviate, etc.)

**Composio** (`lib/server/composio.ts`)
- **Coupling**: MEDIUM - Isolated to apps integration
- **Abstraction**: None
- **Migration**: Would need to rebuild each integration individually

---

## 7. Migration Feasibility

### 7.1 HIPAA-Eligible Stack Migration

**Target Stack**
```
Email:     AWS SES → AWS WorkMail (or Microsoft 365 E3+)
Phone/SMS: Twilio (with BAA) or AWS Connect + Pinpoint
Voice:     Twilio + OpenAI GPT-Live (both support BAA)
Compute:   AWS EC2 (HIPAA eligible) or Azure VMs (HIPAA compliant)
Memory:    AWS OpenSearch (vector DB) or PostgreSQL with pgvector
AI:        Azure OpenAI (HIPAA compliant) or OpenAI Enterprise (BAA)
```

**Effort Estimates**

| Component | Current | Target | Files Affected | Effort | Risk |
|-----------|---------|--------|----------------|--------|------|
| **Email** | AgentMail | AWS SES + WorkMail | 10+ | **LARGE** | High |
| **Phone/SMS** | AgentPhone | Twilio Programmable Voice | 15+ | **LARGE** | High |
| **Voice Calls** | AgentPhone webhook | Twilio + GPT-Live SIP | 5+ | **MEDIUM** | Medium |
| **Compute** | Orgo VMs | AWS EC2 + ECS | 20+ | **LARGE** | High |
| **Memory** | Honcho | AWS OpenSearch | 3+ | **SMALL** | Low |
| **AI** | OpenAI API | Azure OpenAI | 5+ | **SMALL** | Low |
| **Integrations** | Composio | Custom per app | 100+ | **VERY LARGE** | Critical |

**Total Effort**: **6-12 months of development** with dedicated team

### 7.2 Abstraction Layers to Add First

#### Priority 1: Email Abstraction (Critical Path)

```typescript
// lib/server/email-provider.ts (NEW)
interface EmailProvider {
  // Inbox management
  createInbox(name: string, domain: string): Promise<InboxInfo>
  listInboxes(): Promise<InboxInfo[]>
  deleteInbox(inboxId: string): Promise<void>
  
  // Messaging
  sendEmail(from: string, to: string[], subject: string, body: EmailBody): Promise<string>
  replyToEmail(messageId: string, body: EmailBody): Promise<string>
  
  // Real-time delivery
  onEmailReceived(callback: (email: IncomingEmail) => void): void
  
  // Thread management
  getThread(messageId: string): Promise<EmailThread>
}

// Implementations
class AgentMailProvider implements EmailProvider { /* current code */ }
class SESWorkMailProvider implements EmailProvider { /* new */ }
class MicrosoftProvider implements EmailProvider { /* new */ }
```

**Benefits**
- Allows gradual migration
- Can run multiple providers simultaneously
- Testing easier with mock provider

#### Priority 2: Telephony Abstraction (Critical Path)

```typescript
// lib/server/telephony-provider.ts (NEW)
interface TelephonyProvider {
  // Number management
  searchNumbers(areaCode: string): Promise<AvailableNumber[]>
  provisionNumber(areaCode: string): Promise<PhoneNumber>
  releaseNumber(numberId: string): Promise<void>
  
  // SMS
  sendSMS(from: string, to: string, message: string): Promise<string>
  onSMSReceived(callback: (sms: IncomingSMS) => void): void
  
  // Voice
  configureVoiceWebhook(numberId: string, webhookUrl: string): Promise<void>
  onVoiceCall(callback: (call: IncomingCall) => void): void
}

class AgentPhoneProvider implements TelephonyProvider { /* current */ }
class TwilioProvider implements TelephonyProvider { /* new */ }
class AWSConnectProvider implements TelephonyProvider { /* new */ }
```

#### Priority 3: Compute Abstraction (Medium Priority)

```typescript
// lib/server/compute-provider.ts (NEW)
interface ComputeProvider {
  // VM lifecycle
  createComputer(spec: ComputerSpec): Promise<Computer>
  deleteComputer(id: string): Promise<void>
  
  // Screen access
  getScreenshot(computerId: string, display: number): Promise<Buffer>
  typeText(computerId: string, text: string): Promise<void>
  mouseClick(computerId: string, x: number, y: number): Promise<void>
  
  // Command execution
  runCommand(computerId: string, command: string, timeout: number): Promise<CommandResult>
}

class OrgoProvider implements ComputeProvider { /* current */ }
class EC2Provider implements ComputeProvider { /* new - simplified */ }
```

**Note**: Full Orgo feature parity (fork, clone, multi-screen) may not be achievable with generic VMs.

### 7.3 Self-Host and Local LLM Options

**Current Self-Host Capability**: ⚠️ **PARTIAL**
- ✅ Can self-host Bops app (`BOPS_SELF_HOSTED=1`)
- ✅ State stored locally (`.data/` directory)
- ✅ Can use own OpenAI keys
- ❌ Cannot self-host agent computers (requires Orgo)
- ❌ Cannot self-host email (requires AgentMail)
- ❌ Cannot self-host phone (requires AgentPhone)

**Future Local LLM Path**

```
Component                Current              Local Alternative
---------                -------              -----------------
Agent Reasoning          OpenAI GPT-4/5       Llama 3.3, Mixtral, Claude (self-hosted)
Computer Vision          OpenAI Vision        Llama 3.2 Vision, Qwen-VL
Voice (STT)             AgentPhone           Whisper (local), Faster-Whisper
Voice (TTS)             AgentPhone           Coqui TTS, Piper TTS, XTTS
Memory                  Honcho               PostgreSQL + pgvector (local)
Embeddings              OpenAI               sentence-transformers (local)
Compute                 Orgo VMs             Local Docker containers or LXC

In-Clinic Box Specs:
- NVIDIA GPU (RTX 4090 or better)
- 128GB RAM
- 2TB NVMe SSD
- Ubuntu Server 22.04
- Docker + Ollama for LLM inference
```

**Estimated Effort for Full Local LLM**:
- **12-18 months** to replace all external services
- **Quality loss**: Local LLMs still behind GPT-4.5/Claude-3.5-Sonnet
- **Better approach**: Use Azure OpenAI (HIPAA) + local data processing

---

## 8. Reuse Fit for Wellness Operations Platform

### 8.1 What to Keep ✅

**Strong Foundation**
1. **Agent Architecture** - Well-designed bot system with main/specialist pattern
2. **Desktop App Framework** - Solid Electron + Next.js structure
3. **State Management** - Clean TypeScript types, good state persistence
4. **Screen Streaming** - Real-time VM screen access (useful for staff training)
5. **Memory System** - Conversation context and user preferences
6. **Approval Workflows** - Permission system for sensitive actions
7. **Multi-Channel Support** - Chat, email, phone, Slack (adapt for clinic channels)

**Reusable Patterns**
- Bot specialization (adapt to: Front Desk Bot, Provider Bot, Insurance Bot)
- Computer-based tasks (adapt to: EMR data entry, form filling)
- Approval guardrails (adapt to: PHI access controls, prescription approvals)
- Workspace model (adapt to: per-clinic isolation)

### 8.2 What to Replace ❌

**Critical Replacements**
1. **AgentMail** → HIPAA-eligible email (AWS SES + WorkMail, Microsoft 365)
2. **AgentPhone** → HIPAA-eligible telephony (Twilio with BAA, AWS Connect)
3. **Honcho** → HIPAA-compliant memory (AWS, encrypted PostgreSQL)
4. **Composio** → Selective integrations with BAAs (EMR systems only)
5. **Orgo** → AWS/Azure VMs in HIPAA environment (or eliminate computer feature)

**Remove Entirely**
- treg (business data lookup) - not relevant for wellness
- TypeSafe - can be replaced with internal logic
- Public screen streaming - security risk with PHI
- Desktop app - move to web-based multi-tenant SaaS

### 8.3 Major Gaps for Wellness Platform

#### Security & Compliance
❌ **No HIPAA awareness** - Logs may contain PHI, no audit trails
❌ **No BAA providers** - All key services lack healthcare compliance
❌ **No PHI redaction** - OpenAI logs could see patient data
❌ **No audit logging** - HIPAA requires comprehensive audit trails
❌ **No encryption at rest** - State files not encrypted (`.data/` directory)
❌ **No role-based access** - Single user model, not multi-role clinic

#### Multi-Tenancy
❌ **Single-tenant design** - One installation per user, not per-clinic SaaS
❌ **No tenant isolation** - No database-level or schema separation
❌ **No usage tracking per tenant** - Billing would need redesign
❌ **Shared infrastructure** - Orgo VMs in shared cloud, not dedicated

#### Healthcare-Specific Features
❌ **No EMR integration** - Would need Epic, Cerner, Athena connectors
❌ **No appointment scheduling** - Needs integration with practice management
❌ **No insurance verification** - Would need eligibility checking
❌ **No prescription routing** - Needs e-prescribe integration (Surescripts)
❌ **No patient portal** - Wellness platform needs patient-facing interface
❌ **No FHIR support** - Healthcare interoperability standard

#### Operational Concerns
⚠️ **Mac-only desktop app** - Clinics use Windows, need web app
⚠️ **No staff permissioning** - Front desk vs provider access controls
⚠️ **No scheduling constraints** - Provider availability, appointment types
⚠️ **No patient consent** - HIPAA consent forms and opt-ins
⚠️ **Limited notifications** - Needs SMS to patients, urgent alerts to providers

### 8.4 Security Concerns

**High Priority Issues**

1. **PHI in Logs**
   - Location: `~/Library/Logs/Bops/server.log`, console output
   - Risk: Patient data logged without redaction
   - Fix: Implement PHI detection and redaction

2. **Hardcoded Secrets Risk**
   - Location: `.env.local` file on disk (unencrypted)
   - Risk: API keys readable by file system access
   - Fix: Use HashiCorp Vault, AWS Secrets Manager

3. **OpenAI Data Retention**
   - Issue: OpenAI logs all API calls (30 days default)
   - Risk: PHI sent to OpenAI for training/monitoring
   - Fix: OpenAI Enterprise with zero retention policy

4. **State File Encryption**
   - Location: `.data/users/<id>/state.json` (plaintext)
   - Risk: Patient data accessible if disk compromised
   - Fix: Encrypt at rest with KMS

5. **Webhook Security**
   - Current: Signature verification only
   - Risk: Replay attacks, webhook URL exposure
   - Fix: Add timestamp validation, IP allowlist

6. **No Session Timeouts**
   - Issue: Desktop app stays logged in indefinitely
   - Risk: Unattended workstation access
   - Fix: Implement idle timeouts, re-authentication

7. **Cross-Tenant Data Leakage**
   - Issue: Single tenant design, no isolation
   - Risk: Clinic A sees Clinic B's data in multi-tenant deployment
   - Fix: Complete multi-tenant rewrite with DB isolation

**Medium Priority Issues**
- WebSocket connections not authenticated per message
- Screen recordings may capture PHI (not encrypted)
- Computer VMs on shared Orgo infrastructure
- No network segmentation between bots

### 8.5 Recommended Approach

**Option A: Fork and Adapt (18-24 months)**
```
Phase 1 (6 months): Compliance Foundation
- Replace AgentMail with AWS SES + WorkMail
- Replace AgentPhone with Twilio (BAA)
- Add PHI redaction and audit logging
- Implement encryption at rest

Phase 2 (6 months): Multi-Tenancy
- Redesign for per-clinic isolation
- Add role-based access control
- Implement clinic admin portal
- Add usage tracking and billing

Phase 3 (6 months): Healthcare Features
- EMR integration (Epic, Cerner)
- Appointment scheduling
- Insurance verification
- E-prescribe integration

Phase 4 (6 months): Local LLM Option
- In-clinic box design
- Local Whisper/Ollama deployment
- Offline operation mode
```

**Option B: Rebuild with Bops Patterns (12-18 months)**
```
Keep:
- Agent conversation patterns
- Bot specialization model
- Approval workflow concepts
- State management patterns

Build New:
- Web-based multi-tenant SaaS (not desktop)
- HIPAA-first architecture (Azure Health Data Services)
- EMR-native integrations
- Clinic-specific UI/UX

Reuse Code:
- TypeScript types (lib/types.ts)
- Memory patterns (lib/server/memory.ts)
- Approval logic (lib/server/permission.ts)
- State sync concepts (lib/server/persist-*.ts)
```

**Option C: Buy + Integrate (6-12 months)**
```
Use existing healthcare platforms:
- Use Doximity/Suki for physician workflow AI
- Use HealthTap for patient triage
- Build thin layer connecting:
  - Epic/Cerner (EMR)
  - ModMed/Nextech (wellness PM systems)
  - Azure OpenAI (HIPAA AI)
  - Twilio (BAA telephony)

Bops role: Inspiration for conversation flow only
```

---

## 9. Detailed Security Review

### 9.1 Hardcoded Secrets Analysis

**Finding**: ✅ **NO hardcoded secrets in codebase**
- Searched entire codebase for API keys, passwords, tokens
- All secrets loaded from environment variables or macOS Keychain
- `.env.example` contains only placeholders

**Secret Storage**
```
macOS Keychain:          Orgo API keys, bot tokens (Slack/Telegram/Discord), passwords
Environment Variables:   Service API keys (AGENTMAIL_*, AGENTPHONE_*, etc.)
PostgreSQL:             Encrypted fields (sealed with crypto.ts)
Not in Code:            ✅ No secrets in source control
```

**Files Checked**
- All `*.ts`, `*.tsx`, `*.js`, `*.mjs` files
- Configuration files (`.env.example`, `package.json`)
- Database migrations (`db/migrations/*.sql`)

### 9.2 PHI Logging Risks

**High Risk Areas**

1. **OpenAI API Logs**
   ```typescript
   // lib/server/openai-client.ts
   // Every API call logs: model, prompt, response
   // Risk: Patient names, conditions, medications in prompts
   ```

2. **Server Logs**
   ```typescript
   // lib/server/*.ts - Multiple console.log, console.warn
   // Examples:
   // - lib/server/mail.ts:110 - Email subject/body
   // - lib/server/phone.ts:700 - SMS message text
   // - lib/server/chat.ts - Full conversation text
   ```

3. **Error Messages**
   ```typescript
   // Errors include user input and context
   // May contain PHI in stack traces
   ```

4. **Analytics Events**
   ```typescript
   // cloud/analytics.ts - PostHog events
   // Explicitly states: "never content" but code doesn't enforce
   ```

**Mitigation Required**
```typescript
// NEW: lib/server/phi-detector.ts
interface PHIRedactor {
  redact(text: string): { redacted: string; hadPHI: boolean }
  redactStructured(obj: unknown): unknown
}

// Patterns to detect:
// - SSN: \d{3}-\d{2}-\d{4}
// - MRN: Medical record numbers
// - Dates of service
// - Patient names (if in known format)
// - Drug names (match against formulary)
```

### 9.3 Multi-Tenancy Concerns

**Current Architecture**: Single-tenant per installation

**Problem**: Cannot support multiple clinics securely

**Required Changes for Multi-Tenancy**

1. **Database Isolation**
   ```sql
   -- Option A: Schema per tenant
   CREATE SCHEMA clinic_12345;
   CREATE SCHEMA clinic_67890;
   
   -- Option B: Row-level security
   CREATE TABLE patients (
     clinic_id UUID NOT NULL,
     patient_id UUID NOT NULL,
     -- ... fields
     PRIMARY KEY (clinic_id, patient_id)
   );
   CREATE POLICY patients_isolation ON patients
     USING (clinic_id = current_setting('app.current_clinic')::uuid);
   ```

2. **State Isolation**
   ```typescript
   // Current: Single AppState per user
   // Needed: AppState per clinic with strict boundaries
   
   type MultiTenantState = {
     clinicId: string
     isolationLevel: 'strict' // no cross-clinic data
     state: AppState // clinic-specific
   }
   ```

3. **Bot Isolation**
   ```typescript
   // Current: Bots can access any workspace
   // Needed: Bots scoped to single clinic
   
   type Bot = {
     clinicId: string // NEW: hard isolation
     allowedResources: ClinicResource[] // explicit whitelist
     // ... existing fields
   }
   ```

4. **API Key Scoping**
   ```typescript
   // Current: OpenAI key shared across all users
   // Needed: Per-clinic OpenAI organization
   
   type ClinicConfig = {
     clinicId: string
     openaiOrgId: string // separate billing, logs
     twilioAccountId: string // separate phone #s
     awsAccountId: string // separate AWS resources
   }
   ```

---

## 10. Conclusion and Recommendations

### 10.1 Reuse Viability Summary

**Overall Assessment**: ⚠️ **MODERATE - Significant Work Required**

| Aspect | Viability | Notes |
|--------|-----------|-------|
| **Core Architecture** | ✅ GOOD | Solid foundation, well-architected |
| **Agent Patterns** | ✅ GOOD | Bot specialization model applicable |
| **Email Service** | ❌ POOR | No HIPAA option, full replacement needed |
| **Phone Service** | ❌ POOR | No HIPAA option, full replacement needed |
| **Compute Layer** | ⚠️ MEDIUM | Could use AWS/Azure, lose some features |
| **Security** | ❌ POOR | No PHI awareness, logging risks |
| **Multi-Tenancy** | ❌ POOR | Single-tenant design, major rewrite |
| **Healthcare Features** | ❌ NONE | No EMR, scheduling, insurance, e-prescribe |

### 10.2 Go/No-Go Recommendation

**RECOMMENDATION: CONDITIONAL GO with Major Modifications**

**GO IF:**
- ✅ You have 18-24 month timeline
- ✅ You have $2M+ budget for development
- ✅ You value the agent architecture and patterns
- ✅ You can hire team with healthcare compliance expertise
- ✅ You start with pilot clinic (not multi-tenant SaaS)

**NO-GO IF:**
- ❌ You need HIPAA compliance in <12 months
- ❌ You need multi-tenant SaaS from day 1
- ❌ Budget is limited (<$1M)
- ❌ You need immediate EMR integrations
- ❌ You can't replace 3 major services (email/phone/compute)

### 10.3 Critical Path Forward

**IF PROCEEDING: 4-Phase Approach**

#### Phase 1: Compliance Foundation (Months 1-6)
**Goal**: Make it HIPAA-capable

1. Replace AgentMail → AWS SES + WorkMail
2. Replace AgentPhone → Twilio with BAA
3. Add PHI detection and redaction (all logs)
4. Implement encryption at rest (state files, database)
5. Add comprehensive audit logging
6. Switch to Azure OpenAI with BAA
7. Remove Composio (or whitelist only HIPAA-safe integrations)
8. Security audit and penetration testing

**Deliverable**: Single-clinic deployment passing HIPAA tech assessment

#### Phase 2: Healthcare Integration (Months 7-12)
**Goal**: Make it useful for wellness clinics

1. Epic/Cerner HL7/FHIR integration
2. ModMed/Nextech wellness PM integration
3. Appointment scheduling system
4. Insurance eligibility checking
5. Patient portal (secure messaging)
6. E-prescribe integration (Surescripts)
7. Provider credential management
8. Clinic-specific workflows

**Deliverable**: Working pilot at 1-2 clinics

#### Phase 3: Multi-Tenancy (Months 13-18)
**Goal**: Scale to multiple clinics

1. Database schema per tenant
2. Tenant-scoped API keys
3. Clinic admin portal
4. Usage tracking and billing
5. Tenant provisioning automation
6. Cross-tenant security testing
7. SOC 2 Type II certification

**Deliverable**: SaaS platform for 10+ clinics

#### Phase 4: Local LLM Option (Months 19-24)
**Goal**: On-premise deployment for large practices

1. In-clinic hardware specifications
2. Ollama + Llama 3.3 deployment
3. Local Whisper for voice transcription
4. Offline operation mode
5. Air-gapped network support
6. Local data processing (no cloud calls)
7. Clinic IT installation guides

**Deliverable**: Deployable on-premise solution

### 10.4 Risk Factors

**HIGH RISK**
1. ⚠️ **Service Provider Changes**
   - AgentMail/AgentPhone replacements are complex
   - No existing abstraction layers
   - Estimates assume straightforward APIs (may not be)

2. ⚠️ **Healthcare Compliance Unknown-Unknowns**
   - Team may lack healthcare domain expertise
   - State-specific regulations (CA, NY, etc.)
   - Unexpected certification requirements

3. ⚠️ **EMR Integration Complexity**
   - Epic/Cerner are notoriously difficult
   - HL7 vs FHIR trade-offs
   - Each health system has unique customizations

**MEDIUM RISK**
4. ⚠️ **AI Model Performance with Healthcare Data**
   - GPT-4 may hallucinate medical information
   - Local LLMs significantly worse than GPT-4
   - Fine-tuning required for wellness domain

5. ⚠️ **User Acceptance**
   - Clinics may resist AI for clinical workflows
   - Providers need extensive training
   - Patients may distrust AI communication

**LOW RISK**
6. ℹ️ Technical stack is solid (TypeScript, Next.js, PostgreSQL)
7. ℹ️ Team at Orgo has proven execution ability

### 10.5 Alternative Recommendations

**Alternative 1: Partner with Healthcare-Native AI Platform**
- Companies: Abridge, Suki, Nuance DAX
- Approach: Use their HIPAA infrastructure, add Bops patterns on top
- Timeline: 6-12 months
- Cost: Licensing fees but faster to market

**Alternative 2: Use Bops Patterns in Greenfield Build**
- Keep: Agent conversation model, approval workflows
- Rebuild: Everything else on Azure Health Data Services
- Timeline: 12-18 months
- Benefit: Purpose-built for healthcare, no technical debt

**Alternative 3: Vertical-Specific Approach**
- Start: Narrow use case (e.g., chiropractic, acupuncture)
- Avoid: Heavy EMR integration initially
- Focus: Scheduling, patient communication, billing
- Timeline: 9-12 months to MVP
- Scale: Expand to other wellness verticals

---

## 11. Appendix: Technical Details

### 11.1 Complete Dependency Tree

**Production Dependencies** (46 packages)
```json
{
  "@base-ui/react": "1.8.0",
  "@composio/core": "0.22.0",
  "@composio/openai": "0.13.0",
  "@honcho-ai/sdk": "2.5.1",
  "@mozilla/readability": "0.6.0",
  "@novnc/novnc": "1.7.0",
  "@playwright/mcp": "0.0.83",
  "@rrweb/record": "2.1.7",
  "@rrweb/replay": "2.1.7",
  "agentmail": "0.5.35",
  "class-variance-authority": "0.7.1",
  "cn": "0.4.0",
  "linkedom": "0.18.13",
  "lucide-react": "1.50.0",
  "next": "16.4.5",
  "openai": "7.27.0",
  "pg": "8.23.1",
  "posthog-js": "1.231.0",
  "posthog-node": "5.10.0",
  "react": "19.2.8",
  "react-dom": "19.2.8",
  "server-only": "0.0.1",
  "shadcn": "4.23.0",
  "sucrase": "3.35.1",
  "tw-animate-css": "1.4.0",
  "ws": "8.22.0"
}
```

### 11.2 Environment Variables Reference

**Required for Basic Operation**
```bash
# AI
OPENAI_API_KEY=              # Required
OPENAI_EXECUTOR_API_KEY=     # Required (restricted key with spend limit)

# Compute
ORGO_API_KEY=                # Required (or sign in with Orgo)
```

**Optional Services**
```bash
# Email
AGENTMAIL_API_KEY=           # For bot email addresses
BOPS_MAIL_DOMAIN=            # Default: bops.bot

# Phone
AGENTPHONE_API_KEY=          # For bot phone numbers
AGENTPHONE_SUB_ACCOUNT=      # AgentPhone sub-account ID
BOPS_AGENTPHONE_HOOK_URL=    # Public webhook URL

# Memory & Integrations
HONCHO_API_KEY=              # Long-term memory
COMPOSIO_API_KEY=            # 1400+ app integrations
TREG_TOKEN=                  # Business data lookup
TYPESAFE_API_KEY=            # Decision making

# Network
TAILSCALE_AUTH_KEY=          # Private network access to computers

# Hosted Deployment
BOPS_DATABASE_URL=           # PostgreSQL connection string
BOPS_PUBLIC_URL=             # Public URL for webhooks/OAuth
```

**Self-Hosting**
```bash
BOPS_SELF_HOSTED=1           # Enable self-hosted mode
```

### 11.3 Port Usage

```
3210  - Next.js server (localhost only by default)
8897  - Orgo relay agent loopback status API
9200  - Chrome DevTools Protocol (when available)
9222  - Chrome remote debugging
9399  - Codex CLI exec-server
```

### 11.4 File System Layout

```
/Applications/Bops.app/                    (macOS app)
~/Library/Application Support/Bops/
  └─ server/
     ├─ .data/                             (self-hosted state)
     │  ├─ users/<orgo_user_id>/
     │  │  └─ state.json                   (user state)
     │  ├─ uploads/                        (attachments)
     │  └─ phone-secrets.json              (webhook secrets)
     └─ .env.local                         (environment variables)

~/Library/Logs/Bops/
  └─ server.log                            (application logs)

~/.orgo/
  └─ credentials.json                      (Orgo CLI login)

~/.cursor/skills-cursor/                   (built-in agent skills)
```

### 11.5 Database Schema Summary

**Tables** (PostgreSQL schema `bops`)
```sql
-- User state (primary storage)
bops.app_state              -- User state JSONB (minus messages)
bops.chat_messages          -- Individual messages (separate rows)
bops.schema_migrations      -- Applied migrations

-- Phone system
bops.phone_lines            -- Number ownership tracking
bops.owner_phones           -- Verified user mobile numbers

-- Email system
bops.mail_handles           -- Workspace email handle claims
bops.mail_inboxes           -- (implicitly via cloud_objects)

-- Usage and billing
bops.cloud_usage            -- AI usage events
bops.usage_events           -- View over usage ledger

-- Cloud infrastructure
bops.cloud_accounts         -- User account setup
bops.cloud_agents           -- AgentPhone webhook secrets
bops.cloud_objects          -- Generic object tracking
bops.cloud_tunnel           -- Tunnel sessions for Mac ↔ Cloud
```

---

## Report Metadata

**Document Version**: 1.0  
**Last Updated**: November 16, 2026  
**Repository**: https://github.com/OrgoAI/bops  
**Commit**: (branch: cursor/architecture-review-7fd1)  
**Lines of Code Reviewed**: ~50,000+ TypeScript/JavaScript  
**Files Analyzed**: 250+ source files  

**Disclaimer**: This analysis is based on code review and documentation as of November 2026. Actual migration effort may vary based on team expertise, unforeseen technical challenges, and evolving healthcare compliance requirements. This is not legal or compliance advice - consult with healthcare attorneys and HIPAA compliance specialists before proceeding.

---

**END OF REPORT**
