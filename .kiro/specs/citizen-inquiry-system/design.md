# Design Document: Citizen Inquiry System

## Overview

The Citizen Inquiry System is a cloud-native, multi-channel platform built on FedRAMP Moderate authorized AWS services. The architecture prioritizes security, accessibility, and compliance while enabling natural language processing for inquiry categorization and response generation. The system integrates with legacy SOAP-based case management and SharePoint 2016 knowledge base systems through resilient adapter patterns.

### Key Design Principles

1. **Security-First**: All components assume zero trust; encryption everywhere, least privilege access
2. **Accessibility-Native**: WCAG 2.1 AA compliance built into component design, not bolted on
3. **Resilient Integration**: Graceful degradation when legacy systems are unavailable
4. **Audit Everything**: Immutable audit trails for all citizen data interactions
5. **Bilingual by Design**: Spanish/English support at every layer, not translation afterthought

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CITIZEN CHANNELS                                │
├─────────────────┬─────────────────┬─────────────────────────────────────────┤
│   Web Portal    │   Email Intake  │        Phone Transcript Upload          │
│   (React SPA)   │   (SES + Lambda)│           (Staff Dashboard)             │
└────────┬────────┴────────┬────────┴──────────────────┬──────────────────────┘
         │                 │                           │
         ▼                 ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AWS API GATEWAY (REST)                               │
│                    WAF + Rate Limiting + Request Validation                  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION LAYER (ECS Fargate)                      │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│  Inquiry API    │  Auth Service   │  NLP Service    │  Response Service     │
│  (Node.js)      │  (Cognito)      │  (Comprehend)   │  (Bedrock)            │
└────────┬────────┴────────┬────────┴────────┬────────┴───────────┬───────────┘
         │                 │                 │                     │
         ▼                 ▼                 ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA LAYER                                         │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│  Aurora PG      │  S3 (Documents) │  ElastiCache    │  DynamoDB (Audit)     │
│  (Inquiries)    │  (Encrypted)    │  (Sessions)     │  (7-year retention)   │
└─────────────────┴─────────────────┴─────────────────┴───────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      INTEGRATION LAYER                                       │
├─────────────────────────────────┬───────────────────────────────────────────┤
│  Case Management Adapter        │  Knowledge Base Adapter                   │
│  (SOAP Client + SQS Queue)      │  (SharePoint REST + Cache)                │
└─────────────────────────────────┴───────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
┌─────────────────────────────────┐ ┌─────────────────────────────────────────┐
│  Legacy Case Management System  │ │  SharePoint 2016 Knowledge Base         │
│  (On-Premises SOAP API)         │ │  (On-Premises)                          │
└─────────────────────────────────┘ └─────────────────────────────────────────┘
```

### Network Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS GovCloud (US)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         VPC (10.0.0.0/16)                           │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │    │
│  │  │ Public Subnet   │  │ Public Subnet   │  │ Public Subnet   │      │    │
│  │  │ (10.0.1.0/24)   │  │ (10.0.2.0/24)   │  │ (10.0.3.0/24)   │      │    │
│  │  │ AZ-a            │  │ AZ-b            │  │ AZ-c            │      │    │
│  │  │ ALB, NAT GW     │  │ ALB, NAT GW     │  │ ALB, NAT GW     │      │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │    │
│  │  │ Private Subnet  │  │ Private Subnet  │  │ Private Subnet  │      │    │
│  │  │ (10.0.11.0/24)  │  │ (10.0.12.0/24)  │  │ (10.0.13.0/24)  │      │    │
│  │  │ AZ-a            │  │ AZ-b            │  │ AZ-c            │      │    │
│  │  │ ECS Tasks       │  │ ECS Tasks       │  │ ECS Tasks       │      │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │    │
│  │  │ Data Subnet     │  │ Data Subnet     │  │ Data Subnet     │      │    │
│  │  │ (10.0.21.0/24)  │  │ (10.0.22.0/24)  │  │ (10.0.23.0/24)  │      │    │
│  │  │ AZ-a            │  │ AZ-b            │  │ AZ-c            │      │    │
│  │  │ Aurora, Cache   │  │ Aurora, Cache   │  │ Aurora, Cache   │      │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                          AWS Direct Connect                                  │
│                                    │                                         │
└────────────────────────────────────┼─────────────────────────────────────────┘
                                     │
                                     ▼
                    ┌────────────────────────────────┐
                    │   Agency On-Premises Network   │
                    │   - Case Management System     │
                    │   - SharePoint 2016            │
                    └────────────────────────────────┘
```

## Components and Interfaces

### 1. Citizen Portal (Frontend)

**Technology**: React 18 with TypeScript, Tailwind CSS, React Query

**Responsibilities**:
- Render accessible, mobile-responsive inquiry forms
- Handle document uploads with client-side validation
- Display inquiry status and history
- Support English/Spanish language switching
- Implement WCAG 2.1 AA compliance

**Key Interfaces**:
```typescript
interface InquirySubmission {
  submit(inquiry: InquiryFormData): Promise<InquiryConfirmation>;
  uploadDocument(file: File, inquiryId: string): Promise<DocumentReference>;
  getStatus(trackingNumber: string): Promise<InquiryStatus>;
}

interface AccessibilityProvider {
  announceToScreenReader(message: string, priority: 'polite' | 'assertive'): void;
  manageFocus(elementId: string): void;
  getContrastRatio(foreground: string, background: string): number;
}
```

### 2. Staff Dashboard (Frontend)

**Technology**: React 18 with TypeScript, Tailwind CSS, React Query, AG Grid

**Responsibilities**:
- Display inquiry queue with filtering and sorting
- Provide response composition interface
- Generate performance reports
- Manage inquiry assignments and escalations

**Key Interfaces**:
```typescript
interface InquiryManagement {
  getQueue(filters: QueueFilters): Promise<PaginatedInquiries>;
  assignInquiry(inquiryId: string, staffId: string): Promise<void>;
  updateStatus(inquiryId: string, status: InquiryStatus): Promise<void>;
  submitResponse(inquiryId: string, response: ResponseDraft): Promise<void>;
}

interface ReportingService {
  generateReport(params: ReportParams): Promise<ReportData>;
  exportReport(reportId: string, format: 'pdf' | 'csv'): Promise<Blob>;
}
```

### 3. Inquiry API Service

**Technology**: Node.js 20 LTS, Express.js, TypeScript

**Responsibilities**:
- Handle inquiry CRUD operations
- Validate and sanitize input data
- Coordinate with NLP and Response services
- Manage document processing workflow
- Emit audit events for all operations

**Key Interfaces**:
```typescript
interface InquiryService {
  createInquiry(data: CreateInquiryDTO): Promise<Inquiry>;
  getInquiry(id: string, userId: string): Promise<Inquiry>;
  updateInquiry(id: string, data: UpdateInquiryDTO): Promise<Inquiry>;
  listInquiries(filters: InquiryFilters, pagination: PaginationParams): Promise<PaginatedResult<Inquiry>>;
}

interface DocumentProcessor {
  processDocument(file: Buffer, mimeType: string): Promise<ExtractedContent>;
  validateDocument(file: Buffer): Promise<ValidationResult>;
}
```

### 4. Authentication Service

**Technology**: AWS Cognito with custom Lambda triggers

**Responsibilities**:
- Manage citizen and staff authentication
- Enforce MFA for staff users
- Handle session management with 15-minute timeout
- Implement RBAC with four roles

**Key Interfaces**:
```typescript
interface AuthService {
  authenticate(credentials: Credentials): Promise<AuthResult>;
  refreshToken(refreshToken: string): Promise<TokenPair>;
  validateSession(accessToken: string): Promise<SessionInfo>;
  enforceRole(userId: string, requiredRole: Role): Promise<boolean>;
}

type Role = 'citizen' | 'staff' | 'supervisor' | 'admin';
```

### 5. NLP Service

**Technology**: AWS Comprehend, AWS Translate, Custom classification model

**Responsibilities**:
- Detect inquiry language (English/Spanish)
- Categorize inquiries with confidence scores
- Extract key entities and topics
- Flag low-confidence classifications for review

**Key Interfaces**:
```typescript
interface NLPService {
  analyzeInquiry(text: string): Promise<NLPAnalysis>;
  detectLanguage(text: string): Promise<LanguageDetection>;
  translateText(text: string, targetLanguage: 'en' | 'es'): Promise<TranslationResult>;
}

interface NLPAnalysis {
  primaryCategory: Category;
  secondaryCategories: Category[];
  confidenceScore: number;
  entities: Entity[];
  sentiment: Sentiment;
  requiresManualReview: boolean;
}
```

### 6. Response Generation Service

**Technology**: AWS Bedrock (Claude), Custom prompt engineering

**Responsibilities**:
- Query knowledge base for relevant content
- Generate response suggestions with government tone
- Support bilingual response generation
- Log original suggestions vs. final responses

**Key Interfaces**:
```typescript
interface ResponseService {
  generateSuggestions(inquiry: Inquiry, context: KnowledgeContext): Promise<ResponseSuggestion[]>;
  applyToneGuidelines(text: string, language: 'en' | 'es'): Promise<string>;
  translateResponse(text: string, targetLanguage: 'en' | 'es'): Promise<string>;
}

interface ResponseSuggestion {
  id: string;
  content: string;
  relevanceScore: number;
  sourceDocuments: DocumentReference[];
  language: 'en' | 'es';
}
```

### 7. Case Management Adapter

**Technology**: Node.js SOAP client, AWS SQS for queuing

**Responsibilities**:
- Synchronize inquiries with legacy case management
- Handle SOAP API communication
- Queue failed requests for retry
- Implement exponential backoff

**Key Interfaces**:
```typescript
interface CaseManagementAdapter {
  syncInquiry(inquiry: Inquiry): Promise<SyncResult>;
  getSyncStatus(inquiryId: string): Promise<SyncStatus>;
  retryFailedSync(inquiryId: string): Promise<void>;
}

interface SyncResult {
  success: boolean;
  externalCaseId?: string;
  error?: SyncError;
  retryScheduled?: Date;
}
```

### 8. Knowledge Base Adapter

**Technology**: SharePoint REST API client, Redis cache

**Responsibilities**:
- Query SharePoint 2016 for relevant content
- Cache frequently accessed content
- Handle SharePoint authentication
- Provide fallback to cached content when unavailable

**Key Interfaces**:
```typescript
interface KnowledgeBaseAdapter {
  search(query: string, category: Category): Promise<KnowledgeArticle[]>;
  getArticle(articleId: string): Promise<KnowledgeArticle>;
  refreshCache(category?: Category): Promise<void>;
  getHealthStatus(): Promise<AdapterHealth>;
}
```

### 9. Audit Service

**Technology**: AWS DynamoDB with TTL disabled, AWS CloudTrail

**Responsibilities**:
- Create immutable audit log entries
- Redact PII from application logs
- Support audit queries with date range filtering
- Ensure 7-year retention

**Key Interfaces**:
```typescript
interface AuditService {
  logAction(event: AuditEvent): Promise<void>;
  queryAuditLog(filters: AuditFilters): Promise<PaginatedResult<AuditEntry>>;
  redactPII(data: unknown): unknown;
}

interface AuditEvent {
  userId: string;
  action: AuditAction;
  resourceType: string;
  resourceId: string;
  metadata: Record<string, unknown>;
  correlationId: string;
}
```

### 10. Email Intake Service

**Technology**: AWS SES, AWS Lambda, S3

**Responsibilities**:
- Receive and parse incoming emails
- Extract attachments and content
- Create inquiry records from emails
- Send acknowledgment emails

**Key Interfaces**:
```typescript
interface EmailIntakeService {
  processIncomingEmail(rawEmail: Buffer): Promise<Inquiry>;
  sendAcknowledgment(inquiry: Inquiry): Promise<void>;
  extractAttachments(email: ParsedEmail): Promise<Attachment[]>;
}
```

## Data Models

### Core Entities

```typescript
// Inquiry - Central entity for citizen questions
interface Inquiry {
  id: string;                          // UUID v4
  trackingNumber: string;              // Human-readable: CIS-2025-000001
  channel: 'web' | 'email' | 'phone';
  status: InquiryStatus;
  priority: 'low' | 'medium' | 'high' | 'urgent';
  
  // Citizen information
  citizenId: string;                   // Reference to Citizen
  contactEmail: string;
  contactPhone?: string;
  preferredLanguage: 'en' | 'es';
  
  // Content
  subject: string;
  body: string;
  originalLanguage: 'en' | 'es';
  
  // Classification
  primaryCategory: string;
  secondaryCategories: string[];
  nlpConfidenceScore: number;
  requiresManualReview: boolean;
  
  // Assignment
  assignedStaffId?: string;
  escalatedTo?: string;
  
  // Documents
  attachments: DocumentReference[];
  
  // Timestamps
  createdAt: Date;
  updatedAt: Date;
  firstResponseAt?: Date;
  resolvedAt?: Date;
  
  // External references
  externalCaseId?: string;            // Legacy case management ID
  syncStatus: SyncStatus;
}

type InquiryStatus = 
  | 'new'
  | 'pending_review'
  | 'in_progress'
  | 'awaiting_citizen'
  | 'escalated'
  | 'resolved'
  | 'closed';

// Citizen - Registered portal users
interface Citizen {
  id: string;
  email: string;
  phone?: string;
  firstName: string;
  lastName: string;
  preferredLanguage: 'en' | 'es';
  createdAt: Date;
  lastLoginAt?: Date;
  
  // Data protection
  dataExportRequestedAt?: Date;
  deletionRequestedAt?: Date;
}

// Staff - Agency employees
interface Staff {
  id: string;
  cognitoUserId: string;
  email: string;
  firstName: string;
  lastName: string;
  role: 'staff' | 'supervisor' | 'admin';
  department: string;
  categories: string[];               // Categories they can handle
  isActive: boolean;
  createdAt: Date;
  lastLoginAt?: Date;
}

// Response - Staff responses to inquiries
interface Response {
  id: string;
  inquiryId: string;
  staffId: string;
  
  // Content
  content: string;
  language: 'en' | 'es';
  
  // AI assistance tracking
  suggestedResponseId?: string;
  originalSuggestion?: string;
  wasEdited: boolean;
  
  // Delivery
  sentAt: Date;
  deliveryChannel: 'email' | 'portal';
  deliveryStatus: 'pending' | 'sent' | 'delivered' | 'failed';
  
  createdAt: Date;
}

// Document - Uploaded files
interface Document {
  id: string;
  inquiryId: string;
  
  // Storage
  s3Key: string;
  s3Bucket: string;
  
  // Metadata
  originalFilename: string;
  mimeType: string;
  sizeBytes: number;
  
  // Processing
  extractedText?: string;
  processingStatus: 'pending' | 'processing' | 'completed' | 'failed';
  
  // Security
  encryptionKeyId: string;
  virusScanStatus: 'pending' | 'clean' | 'infected';
  
  uploadedAt: Date;
  uploadedBy: string;
}

// AuditEntry - Immutable audit log
interface AuditEntry {
  id: string;                         // UUID v4
  timestamp: Date;
  
  // Actor
  userId: string;
  userRole: Role;
  userIp: string;
  
  // Action
  action: AuditAction;
  resourceType: string;
  resourceId: string;
  
  // Context
  correlationId: string;
  sessionId: string;
  
  // Details (PII redacted)
  metadata: Record<string, unknown>;
  
  // Retention
  retentionExpiresAt: Date;           // 7 years from creation
}

type AuditAction =
  | 'inquiry.create'
  | 'inquiry.view'
  | 'inquiry.update'
  | 'inquiry.assign'
  | 'inquiry.escalate'
  | 'response.create'
  | 'response.send'
  | 'document.upload'
  | 'document.view'
  | 'document.download'
  | 'citizen.data_export'
  | 'citizen.data_delete'
  | 'auth.login'
  | 'auth.logout'
  | 'auth.mfa_challenge'
  | 'auth.session_timeout'
  | 'admin.role_change'
  | 'admin.config_change';

// KnowledgeArticle - Cached from SharePoint
interface KnowledgeArticle {
  id: string;
  sharePointId: string;
  
  title: string;
  content: string;
  summary: string;
  
  category: string;
  tags: string[];
  language: 'en' | 'es';
  
  lastModified: Date;
  cachedAt: Date;
  cacheExpiresAt: Date;
}

// SyncQueue - Failed synchronization tracking
interface SyncQueueItem {
  id: string;
  inquiryId: string;
  
  operation: 'create' | 'update';
  payload: Record<string, unknown>;
  
  attempts: number;
  lastAttemptAt: Date;
  nextAttemptAt: Date;
  
  lastError?: string;
  
  createdAt: Date;
  expiresAt: Date;                    // 24 hours from creation
}
```

### Database Schema (PostgreSQL/Aurora)

```sql
-- Inquiries table
CREATE TABLE inquiries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tracking_number VARCHAR(20) UNIQUE NOT NULL,
    channel VARCHAR(10) NOT NULL CHECK (channel IN ('web', 'email', 'phone')),
    status VARCHAR(20) NOT NULL DEFAULT 'new',
    priority VARCHAR(10) NOT NULL DEFAULT 'medium',
    
    citizen_id UUID NOT NULL REFERENCES citizens(id),
    contact_email VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(20),
    preferred_language CHAR(2) NOT NULL DEFAULT 'en',
    
    subject VARCHAR(500) NOT NULL,
    body TEXT NOT NULL,
    original_language CHAR(2) NOT NULL,
    
    primary_category VARCHAR(100),
    secondary_categories TEXT[],
    nlp_confidence_score DECIMAL(3,2),
    requires_manual_review BOOLEAN DEFAULT false,
    
    assigned_staff_id UUID REFERENCES staff(id),
    escalated_to UUID REFERENCES staff(id),
    
    external_case_id VARCHAR(100),
    sync_status VARCHAR(20) DEFAULT 'pending',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    first_response_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ
);

-- Indexes for common queries
CREATE INDEX idx_inquiries_status ON inquiries(status);
CREATE INDEX idx_inquiries_assigned_staff ON inquiries(assigned_staff_id);
CREATE INDEX idx_inquiries_citizen ON inquiries(citizen_id);
CREATE INDEX idx_inquiries_created_at ON inquiries(created_at);
CREATE INDEX idx_inquiries_tracking_number ON inquiries(tracking_number);

-- Citizens table
CREATE TABLE citizens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cognito_user_id VARCHAR(100) UNIQUE,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    preferred_language CHAR(2) NOT NULL DEFAULT 'en',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ,
    
    data_export_requested_at TIMESTAMPTZ,
    deletion_requested_at TIMESTAMPTZ
);

-- Staff table
CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cognito_user_id VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL CHECK (role IN ('staff', 'supervisor', 'admin')),
    department VARCHAR(100) NOT NULL,
    categories TEXT[],
    is_active BOOLEAN DEFAULT true,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ
);

-- Responses table
CREATE TABLE responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inquiry_id UUID NOT NULL REFERENCES inquiries(id),
    staff_id UUID NOT NULL REFERENCES staff(id),
    
    content TEXT NOT NULL,
    language CHAR(2) NOT NULL,
    
    suggested_response_id UUID,
    original_suggestion TEXT,
    was_edited BOOLEAN DEFAULT false,
    
    sent_at TIMESTAMPTZ,
    delivery_channel VARCHAR(10) NOT NULL,
    delivery_status VARCHAR(20) DEFAULT 'pending',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Documents table
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inquiry_id UUID NOT NULL REFERENCES inquiries(id),
    
    s3_key VARCHAR(500) NOT NULL,
    s3_bucket VARCHAR(100) NOT NULL,
    
    original_filename VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    size_bytes BIGINT NOT NULL,
    
    extracted_text TEXT,
    processing_status VARCHAR(20) DEFAULT 'pending',
    
    encryption_key_id VARCHAR(100) NOT NULL,
    virus_scan_status VARCHAR(20) DEFAULT 'pending',
    
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uploaded_by UUID NOT NULL
);
```



## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

Based on the acceptance criteria analysis, the following correctness properties must be validated through property-based testing:

### Property 1: Tracking Number Uniqueness
*For any* set of inquiry submissions through the web portal, each created inquiry SHALL have a unique tracking number that does not collide with any other tracking number in the system.
**Validates: Requirements 1.1**

### Property 2: Channel Intake Round-Trip
*For any* valid inquiry content submitted via email or phone transcript, the created inquiry record SHALL contain the original content, contact information, and channel identifier matching the input.
**Validates: Requirements 1.2, 1.3**

### Property 3: Document Association Integrity
*For any* document uploaded with an inquiry, the document SHALL be retrievable via the inquiry reference and the inquiry SHALL list the document in its attachments.
**Validates: Requirements 1.4**

### Property 4: Validation Preserves Completed Fields
*For any* inquiry form submission with at least one missing required field, the validation response SHALL identify all missing fields AND all previously completed fields SHALL retain their values.
**Validates: Requirements 1.5**

### Property 5: NLP Categorization Completeness
*For any* inquiry text, the NLP Engine SHALL return a primary category and a confidence score between 0 and 1.
**Validates: Requirements 2.1**

### Property 6: Low Confidence Flagging
*For any* inquiry where NLP confidence score is below 0.70, the inquiry SHALL be flagged for manual review (requiresManualReview = true).
**Validates: Requirements 2.2**

### Property 7: Multi-Topic Category Identification
*For any* inquiry containing multiple distinct topics, the NLP analysis SHALL return at least two categories in the secondaryCategories array.
**Validates: Requirements 2.3**

### Property 8: Response Suggestion Constraints
*For any* categorized inquiry, the response suggestions SHALL contain at most 5 items AND each suggestion SHALL have a relevanceScore AND suggestions SHALL be ordered by relevanceScore descending.
**Validates: Requirements 3.1**

### Property 9: Response Language Matching
*For any* inquiry in Spanish (originalLanguage = 'es'), all generated response suggestions SHALL have language = 'es'.
**Validates: Requirements 3.3**

### Property 10: Response Edit Audit Trail
*For any* response where staff edited a suggested response, the Response record SHALL contain both the originalSuggestion and the final content, with wasEdited = true.
**Validates: Requirements 3.5**

### Property 11: Sync Queue on Failure
*For any* inquiry creation when the Case Management System is unavailable, a SyncQueueItem SHALL be created with the inquiry data and nextAttemptAt scheduled according to exponential backoff.
**Validates: Requirements 4.2**

### Property 12: Cache Fallback on KB Unavailability
*For any* knowledge base query when SharePoint is unavailable, the system SHALL return cached articles if available AND the response SHALL indicate degraded status.
**Validates: Requirements 4.4**

### Property 13: Form Error Accessibility
*For any* form validation error, the error message element SHALL be programmatically associated with its input field via aria-describedby or aria-errormessage.
**Validates: Requirements 5.4**

### Property 14: Status Indicator Non-Color Alternatives
*For any* status indicator displayed in the UI, there SHALL exist either a text label or an icon that conveys the status meaning independent of color.
**Validates: Requirements 5.5**

### Property 15: Color Contrast Compliance
*For any* text element in the Citizen Portal, the contrast ratio between text color and background color SHALL be at least 4.5:1 for normal text (< 18pt) and 3:1 for large text (>= 18pt).
**Validates: Requirements 5.6**

### Property 16: RBAC Permission Enforcement
*For any* user attempting an action, the system SHALL permit the action if and only if the user's role includes permission for that action type. Denied actions SHALL create an audit log entry.
**Validates: Requirements 6.3, 6.4**

### Property 17: Session Timeout Enforcement
*For any* user session that has been inactive for 15 minutes or more, subsequent requests SHALL be rejected with an authentication error requiring re-login.
**Validates: Requirements 6.2**

### Property 18: MFA Requirement for Staff
*For any* authentication attempt by a user with role staff, supervisor, or admin, the authentication flow SHALL require MFA completion before granting access.
**Validates: Requirements 6.1**

### Property 19: Audit Log Completeness
*For any* action performed on citizen data (create, read, update, delete), an AuditEntry SHALL be created containing userId, timestamp, action type, resourceType, and resourceId.
**Validates: Requirements 7.1**

### Property 20: PII Redaction in Logs
*For any* log entry written to application logs, PII fields (email, phone, SSN, name, address) SHALL be replaced with redaction markers.
**Validates: Requirements 7.3**

### Property 21: Data Export Completeness
*For any* citizen data export request, the export SHALL contain all inquiries, responses, and documents associated with that citizen.
**Validates: Requirements 8.2**

### Property 22: Data Deletion with Audit Preservation
*For any* citizen data deletion request, PII SHALL be removed from the Citizen and Inquiry records AND associated AuditEntry records SHALL be preserved with anonymized user references.
**Validates: Requirements 8.3**

### Property 23: Graceful Degradation on Legacy Unavailability
*For any* inquiry submission when legacy systems are unavailable, the inquiry SHALL be accepted and stored locally AND the user SHALL receive a status message indicating limited functionality.
**Validates: Requirements 9.4**

### Property 24: Staff Queue Accuracy
*For any* staff member viewing their dashboard, the displayed queue SHALL contain exactly the inquiries where assignedStaffId matches their ID, with correct priority and aging metrics.
**Validates: Requirements 10.1**

### Property 25: Report Metric Accuracy
*For any* performance report request, the calculated metrics (average response time, resolution rate, category distribution) SHALL match the aggregated values from the underlying inquiry data.
**Validates: Requirements 10.2**

### Property 26: Report Filter Correctness
*For any* report with filters applied, all returned records SHALL satisfy all filter criteria (date range, category, staff member, resolution status).
**Validates: Requirements 10.5**

### Property 27: Spanish UI Completeness
*For any* page in the Citizen Portal when preferredLanguage = 'es', all UI labels, buttons, and system messages SHALL be rendered in Spanish.
**Validates: Requirements 11.1**

### Property 28: Spanish Inquiry Processing
*For any* inquiry submitted in Spanish, the system SHALL process it without language conversion AND responses SHALL be generated in Spanish.
**Validates: Requirements 11.2**

### Property 29: Language Detection from Browser
*For any* first visit to the Citizen Portal, the suggested language SHALL match the browser's Accept-Language header preference if it is 'en' or 'es'.
**Validates: Requirements 11.4**

### Property 30: Form Data Preservation on Language Switch
*For any* language switch action while a form has entered data, all form field values SHALL be preserved after the language change.
**Validates: Requirements 11.5**

### Property 31: Touch Target Minimum Size
*For any* interactive element (button, link, input) in the mobile viewport, the element's clickable area SHALL be at least 44x44 CSS pixels.
**Validates: Requirements 12.2**

### Property 32: Input Type Correctness
*For any* form input field, the HTML input type attribute SHALL match the expected data type (email for email fields, tel for phone fields, number for numeric fields).
**Validates: Requirements 12.3**

## Error Handling

### Error Categories and Responses

| Category | HTTP Status | User Message | Logging | Recovery |
|----------|-------------|--------------|---------|----------|
| Validation Error | 400 | Specific field errors | Debug level, no PII | Client retry with corrections |
| Authentication Error | 401 | "Please sign in again" | Info level with user ID | Redirect to login |
| Authorization Error | 403 | "You don't have permission" | Warning level, audit log | Contact administrator |
| Not Found | 404 | "Resource not found" | Debug level | Check URL/ID |
| Rate Limited | 429 | "Too many requests, please wait" | Warning level | Exponential backoff |
| Legacy System Unavailable | 503 | "Some features temporarily limited" | Error level, alert | Queue and retry |
| Internal Error | 500 | "Something went wrong" | Error level with stack trace | Automatic retry, alert |

### Circuit Breaker Configuration

```typescript
interface CircuitBreakerConfig {
  caseManagement: {
    failureThreshold: 5;        // Open after 5 failures
    successThreshold: 3;        // Close after 3 successes
    timeout: 30000;             // 30 second timeout
    resetTimeout: 60000;        // Try again after 1 minute
  };
  knowledgeBase: {
    failureThreshold: 3;
    successThreshold: 2;
    timeout: 10000;
    resetTimeout: 30000;
  };
  nlpService: {
    failureThreshold: 5;
    successThreshold: 3;
    timeout: 5000;
    resetTimeout: 30000;
  };
}
```

### Retry Strategy

```typescript
interface RetryConfig {
  maxAttempts: 3;
  baseDelay: 1000;              // 1 second
  maxDelay: 30000;              // 30 seconds
  backoffMultiplier: 2;         // Exponential backoff
  retryableErrors: [
    'ECONNRESET',
    'ETIMEDOUT',
    'ECONNREFUSED',
    'ServiceUnavailable',
    'ThrottlingException'
  ];
}
```

### Graceful Degradation Modes

| Dependency | Degraded Behavior |
|------------|-------------------|
| Case Management System | Accept inquiries locally, queue sync, show "limited sync" status |
| SharePoint Knowledge Base | Serve cached content, disable new KB searches, show "cached results" |
| NLP Service | Accept inquiries without categorization, flag all for manual review |
| Email Service | Queue outbound emails, show "email pending" status |
| Bedrock Response Generation | Disable AI suggestions, staff compose manually |

## Testing Strategy

### Dual Testing Approach

This system requires both unit testing and property-based testing to ensure correctness:

- **Unit Tests**: Verify specific examples, edge cases, and integration points
- **Property-Based Tests**: Verify universal properties that should hold across all valid inputs

### Property-Based Testing Framework

**Library**: fast-check (TypeScript)

**Configuration**:
```typescript
import fc from 'fast-check';

const defaultConfig: fc.Parameters<unknown> = {
  numRuns: 100,                    // Minimum 100 iterations per property
  verbose: true,
  seed: Date.now(),
  endOnFailure: true,
  markInterruptAsFailure: true
};
```

**Test Annotation Format**:
Each property-based test MUST include a comment linking to the design document:
```typescript
/**
 * **Feature: citizen-inquiry-system, Property 1: Tracking Number Uniqueness**
 * For any set of inquiry submissions, each tracking number must be unique.
 */
```

### Test Categories

#### 1. Data Model Tests
- Inquiry creation and validation
- Document association integrity
- Audit entry completeness
- PII redaction verification

#### 2. NLP Service Tests
- Categorization completeness
- Confidence score bounds
- Multi-topic detection
- Language detection accuracy

#### 3. Response Generation Tests
- Suggestion count limits
- Relevance ordering
- Language matching
- Edit audit trail

#### 4. Integration Adapter Tests
- Sync queue creation on failure
- Retry scheduling with backoff
- Cache fallback behavior
- Circuit breaker state transitions

#### 5. Authentication/Authorization Tests
- RBAC permission matrix
- Session timeout enforcement
- MFA flow completion
- Audit logging for denied actions

#### 6. Accessibility Tests
- Form error associations
- Color contrast ratios
- Touch target sizes
- Input type correctness

#### 7. Localization Tests
- Spanish UI completeness
- Language detection
- Form data preservation on switch

### Generator Strategies

```typescript
// Inquiry content generator
const inquiryContentArb = fc.record({
  subject: fc.string({ minLength: 1, maxLength: 500 }),
  body: fc.string({ minLength: 10, maxLength: 10000 }),
  language: fc.constantFrom('en', 'es'),
  channel: fc.constantFrom('web', 'email', 'phone')
});

// PII data generator for redaction testing
const piiDataArb = fc.record({
  email: fc.emailAddress(),
  phone: fc.stringMatching(/^\d{3}-\d{3}-\d{4}$/),
  ssn: fc.stringMatching(/^\d{3}-\d{2}-\d{4}$/),
  name: fc.string({ minLength: 2, maxLength: 50 })
});

// Role and permission generator
const roleArb = fc.constantFrom('citizen', 'staff', 'supervisor', 'admin');
const actionArb = fc.constantFrom(
  'inquiry.create', 'inquiry.view', 'inquiry.update',
  'inquiry.assign', 'response.create', 'admin.config_change'
);

// Confidence score generator
const confidenceScoreArb = fc.double({ min: 0, max: 1, noNaN: true });

// Form data generator for preservation testing
const formDataArb = fc.dictionary(
  fc.string({ minLength: 1, maxLength: 20 }),
  fc.oneof(fc.string(), fc.integer(), fc.boolean())
);
```

### Test Execution

**Unit Tests**: Jest with TypeScript
```bash
npm run test:unit
```

**Property Tests**: Jest with fast-check
```bash
npm run test:property
```

**All Tests**:
```bash
npm run test
```

### Coverage Requirements

- Unit test coverage: 80% line coverage minimum
- Property tests: All 32 correctness properties implemented
- Integration tests: All external adapter interfaces covered
- Accessibility tests: All WCAG 2.1 AA criteria verified

### Continuous Integration

Property-based tests run on every PR with:
- 100 iterations in CI (fast feedback)
- 1000 iterations in nightly builds (thorough coverage)
- Seed logging for reproducibility of failures
