# Design Document: Clinical Intervention Agent

## Overview

The Clinical Intervention Agent is a Python-based AI decision support system deployed on Amazon Bedrock AgentCore Runtime. It analyzes multi-modal patient data (labs, vitals, clinical notes) using Claude 3.5 Sonnet to generate risk assessments with explainable reasoning.

The system architecture follows a secure, HIPAA-compliant design with all components deployed in a private VPC. Patient data flows through FHIR R4 compliant APIs, is analyzed by the foundation model with reference to a curated knowledge base, and produces structured outputs with full audit trails.

Key design principles:
- **Security First**: All data encrypted at rest and in transit, no PHI in logs
- **Simplicity**: Minimal business logic, leverage managed AWS services
- **Auditability**: Complete traceability of all predictions
- **Interoperability**: FHIR R4 compliance for healthcare integration
- **Explainability**: Clear reasoning with evidence citations

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Private VPC                              │
│                                                                   │
│  ┌──────────────┐      ┌─────────────────────────────────┐     │
│  │   FHIR R4    │      │  Clinical Intervention Agent    │     │
│  │   API Layer  │─────▶│  (AgentCore Runtime)            │     │
│  └──────────────┘      └─────────────────────────────────┘     │
│                                    │                             │
│                         ┌──────────┼──────────┐                 │
│                         │          │          │                 │
│                         ▼          ▼          ▼                 │
│              ┌──────────────┐  ┌─────────┐  ┌──────────────┐   │
│              │   Bedrock    │  │  S3     │  │  PostgreSQL  │   │
│              │  Knowledge   │  │ (Docs)  │  │  (Metadata)  │   │
│              │     Base     │  │         │  │              │   │
│              │ (OpenSearch) │  │         │  │              │   │
│              └──────────────┘  └─────────┘  └──────────────┘   │
│                         │                                        │
│                         ▼                                        │
│              ┌──────────────────┐                               │
│              │  Amazon Bedrock  │                               │
│              │  Claude 3.5      │                               │
│              │  Sonnet          │                               │
│              └──────────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**FHIR R4 API Layer**
- Validates incoming patient data against FHIR R4 schemas
- Transforms FHIR resources into internal representation
- Returns results in FHIR-compliant format
- Handles authentication and authorization

**Clinical Intervention Agent (AgentCore Runtime)**
- Orchestrates the analysis workflow
- Invokes Claude 3.5 Sonnet via Bedrock API
- Queries Knowledge Base for clinical references
- Generates structured risk assessments
- Creates audit logs for all predictions

**Bedrock Knowledge Base (OpenSearch)**
- Stores clinical guidelines and medical references
- Provides semantic search over clinical knowledge
- Returns relevant evidence for risk assessments
- Uses Titan Text Embeddings for vector search

**S3 Document Storage**
- Stores clinical documents with AES-256 encryption
- Maintains versioning for audit purposes
- Enforces bucket policies for VPC-only access

**PostgreSQL Metadata Store**
- Stores audit logs with complete prediction traceability
- Maintains patient consent and opt-out records
- Tracks model invocations and performance metrics
- Encrypted at rest with AWS RDS encryption

**Amazon Bedrock Foundation Models**
- Claude 3.5 Sonnet: Primary reasoning model
- Claude 3 Haiku: Optional lightweight model
- Llama 3.1 70B Instruct: Optional alternative model
- Titan Text Embeddings: Vector embeddings for knowledge base

### Data Flow

1. **Input**: FHIR R4 patient data (Observation, Patient, DiagnosticReport resources) arrives at API
2. **Validation**: API validates FHIR schemas and extracts labs, vitals, clinical notes
3. **Context Retrieval**: Agent queries Knowledge Base for relevant clinical guidelines
4. **Analysis**: Agent constructs prompt with patient data and context, invokes Claude 3.5 Sonnet
5. **Structured Output**: Model returns JSON with risk_score, confidence_score, and three reasons
6. **Audit**: Complete prediction details logged to PostgreSQL
7. **Response**: Results formatted as FHIR R4 response and returned to client

### Security Architecture

**Network Security**
- All components in private VPC subnets
- No direct internet access
- VPC endpoints for AWS services (Bedrock, S3, RDS)
- Security groups restrict traffic between components

**Data Encryption**
- At rest: AES-256 for S3, RDS encryption for PostgreSQL
- In transit: TLS 1.2+ for all connections
- Bedrock API calls use AWS SigV4 signing

**Access Control**
- IAM roles with least privilege for agent execution
- Resource-based policies on S3 buckets
- RDS authentication via IAM
- No hardcoded credentials

**Privacy Controls**
- PHI/PII scrubbing before logging
- Patient opt-out tracking in PostgreSQL
- Data retention policies enforced
- No data sent to external model training

## Components and Interfaces

### 1. FHIR API Handler

**Purpose**: Expose FHIR R4 compliant endpoints for patient data submission and result retrieval

**Interface**:
```python
class FHIRAPIHandler:
    def validate_fhir_bundle(self, bundle: dict) -> ValidationResult:
        """Validate FHIR R4 bundle against schemas"""
        
    def extract_patient_data(self, bundle: dict) -> PatientData:
        """Extract labs, vitals, notes from FHIR resources"""
        
    def format_response(self, assessment: RiskAssessment) -> dict:
        """Format risk assessment as FHIR R4 response"""
```

**Responsibilities**:
- FHIR R4 schema validation
- Resource extraction (Observation, Patient, DiagnosticReport)
- Response formatting
- Error handling for malformed requests

### 2. Clinical Intervention Agent

**Purpose**: Core orchestration logic for risk assessment workflow

**Interface**:
```python
class ClinicalInterventionAgent:
    def analyze_patient(self, patient_data: PatientData) -> RiskAssessment:
        """Main entry point for patient analysis"""
        
    def retrieve_clinical_context(self, patient_data: PatientData) -> List[Reference]:
        """Query knowledge base for relevant guidelines"""
        
    def construct_prompt(self, patient_data: PatientData, context: List[Reference]) -> str:
        """Build prompt for foundation model"""
        
    def invoke_model(self, prompt: str) -> ModelResponse:
        """Call Claude 3.5 Sonnet via Bedrock API"""
        
    def parse_structured_output(self, response: ModelResponse) -> RiskAssessment:
        """Extract risk_score, confidence_score, reasons from model output"""
        
    def create_audit_log(self, patient_data: PatientData, assessment: RiskAssessment, 
                        prompt: str, model_response: ModelResponse, 
                        references: List[Reference]) -> None:
        """Log complete prediction details"""
```

**Responsibilities**:
- Workflow orchestration
- Knowledge base queries
- Prompt engineering
- Model invocation
- Output parsing and validation
- Audit logging

### 3. Knowledge Base Client

**Purpose**: Interface to Bedrock Knowledge Base for clinical reference retrieval

**Interface**:
```python
class KnowledgeBaseClient:
    def semantic_search(self, query: str, top_k: int = 5) -> List[Reference]:
        """Search for relevant clinical guidelines"""
        
    def get_reference_by_id(self, reference_id: str) -> Reference:
        """Retrieve specific reference document"""
```

**Responsibilities**:
- Semantic search over clinical guidelines
- Reference retrieval with citations
- Vector embedding via Titan Text Embeddings

### 4. Bedrock Model Client

**Purpose**: Wrapper for Amazon Bedrock API calls

**Interface**:
```python
class BedrockModelClient:
    def invoke_claude_sonnet(self, prompt: str, system_prompt: str = None, 
                            temperature: float = 0.0) -> ModelResponse:
        """Invoke Claude 3.5 Sonnet with structured output"""
        
    def invoke_with_retry(self, prompt: str, max_retries: int = 3) -> ModelResponse:
        """Invoke model with exponential backoff retry"""
```

**Responsibilities**:
- Bedrock API invocation
- Error handling and retries
- Response parsing
- Token usage tracking

### 5. Audit Logger

**Purpose**: Persist complete audit trails to PostgreSQL

**Interface**:
```python
class AuditLogger:
    def log_prediction(self, prediction_id: str, patient_id: str, 
                      input_data: dict, prompt: str, model_output: dict,
                      references: List[str], assessment: RiskAssessment,
                      timestamp: datetime) -> None:
        """Create audit log entry for prediction"""
        
    def query_audit_logs(self, filters: dict) -> List[AuditLog]:
        """Retrieve audit logs for compliance review"""
```

**Responsibilities**:
- Audit log persistence
- PHI/PII scrubbing before logging
- Query interface for administrators
- Retention policy enforcement

### 6. Patient Consent Manager

**Purpose**: Track patient opt-out preferences

**Interface**:
```python
class PatientConsentManager:
    def check_consent(self, patient_id: str) -> bool:
        """Check if patient has opted out"""
        
    def record_opt_out(self, patient_id: str, timestamp: datetime) -> None:
        """Record patient opt-out request"""
```

**Responsibilities**:
- Consent status tracking
- Opt-out enforcement
- Compliance reporting

## Data Models

### PatientData

```python
@dataclass
class PatientData:
    patient_id: str
    labs: List[LabResult]
    vitals: List[VitalSign]
    clinical_notes: List[ClinicalNote]
    timestamp: datetime
```

### LabResult

```python
@dataclass
class LabResult:
    test_name: str
    value: float
    unit: str
    reference_range: Tuple[float, float]
    timestamp: datetime
    abnormal_flag: Optional[str]  # "H" for high, "L" for low
```

### VitalSign

```python
@dataclass
class VitalSign:
    vital_type: str  # "heart_rate", "blood_pressure", "temperature", etc.
    value: Union[float, str]  # str for blood pressure "120/80"
    unit: str
    timestamp: datetime
```

### ClinicalNote

```python
@dataclass
class ClinicalNote:
    note_type: str  # "progress_note", "admission_note", etc.
    content: str
    author: str
    timestamp: datetime
```

### RiskAssessment

```python
@dataclass
class RiskAssessment:
    prediction_id: str
    patient_id: str
    risk_score: float  # 0.0 to 1.0
    confidence_score: float  # 0.0 to 1.0
    reasons: List[ClinicalReason]  # Exactly 3 reasons
    low_confidence_warning: bool  # True if confidence_score < 0.75
    timestamp: datetime
```

### ClinicalReason

```python
@dataclass
class ClinicalReason:
    reason_text: str
    evidence_sources: List[str]  # References to knowledge base entries
    data_points: List[str]  # Specific patient data supporting this reason
```

### Reference

```python
@dataclass
class Reference:
    reference_id: str
    title: str
    content: str
    source: str  # e.g., "CDC Guidelines", "UpToDate"
    relevance_score: float
```

### AuditLog

```python
@dataclass
class AuditLog:
    log_id: str
    prediction_id: str
    patient_id: str
    input_data: dict  # Serialized PatientData
    prompt: str
    model_output: dict
    references: List[str]
    assessment: dict  # Serialized RiskAssessment
    timestamp: datetime
    model_name: str
    model_version: str
```

### ModelResponse

```python
@dataclass
class ModelResponse:
    content: str  # Raw model output
    stop_reason: str
    usage: dict  # Token counts
    model_id: str
```

### ValidationResult

```python
@dataclass
class ValidationResult:
    is_valid: bool
    errors: List[str]
    warnings: List[str]
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified the following redundancies:
- Properties 3.2-3.5 (audit log content) can be combined into a single comprehensive property about audit log completeness
- Property 13.2 duplicates property 3.1 (audit log creation)
- Property 13.5 duplicates property 3.6 (audit log retrieval)
- Properties 5.1-5.3 (FHIR acceptance) can be combined into one property about FHIR R4 validation

The following properties represent the unique, testable correctness requirements:

### Property 1: Risk Score Bounds

*For any* patient data input, the returned risk_score should be between 0.0 and 1.0 inclusive.

**Validates: Requirements 1.1**

### Property 2: Confidence Score Bounds

*For any* patient data input, the returned confidence_score should be between 0.0 and 1.0 inclusive.

**Validates: Requirements 1.2**

### Property 3: Exactly Three Reasons

*For any* patient data input, the returned reasons array should contain exactly 3 clinical reasons.

**Validates: Requirements 1.3**

### Property 4: Low Confidence Warning

*For any* risk assessment where confidence_score < 0.75, the low_confidence_warning flag should be True.

**Validates: Requirements 1.4**

### Property 5: Audit Log Creation

*For any* prediction generated by the system, an audit log entry should be created in the database.

**Validates: Requirements 3.1, 13.2**

### Property 6: Audit Log Completeness

*For any* audit log entry, it should contain all of: input_data, prompt, model_output, references, and assessment.

**Validates: Requirements 3.2, 3.3, 3.4, 3.5**

### Property 7: Audit Log Retrieval

*For any* prediction_id that has been logged, querying the audit logs should return the complete audit log entry.

**Validates: Requirements 3.6, 13.5**

### Property 8: Opt-Out Enforcement

*For any* patient_id with opt-out status set to True, attempting to process their data should be rejected.

**Validates: Requirements 4.4**

### Property 9: PHI Exclusion from Logs

*For any* log entry created by the system, it should not contain patient identifiers, names, or other PHI/PII.

**Validates: Requirements 4.5**

### Property 10: FHIR R4 Validation

*For any* valid FHIR R4 bundle containing Observation, Patient, or DiagnosticReport resources, the validation should pass and data should be extracted successfully.

**Validates: Requirements 5.1, 5.2, 5.3**

### Property 11: Knowledge Base Query Success

*For any* patient data analysis, the system should successfully query the knowledge base and retrieve at least one clinical reference.

**Validates: Requirements 6.4**

### Property 12: FHIR R4 Response Compliance

*For any* risk assessment, the formatted API response should be valid according to FHIR R4 schemas.

**Validates: Requirements 8.1, 8.3**

### Property 13: FHIR R4 Schema Validation

*For any* FHIR R4 compliant input, validation should pass; for any malformed input, validation should fail with descriptive errors.

**Validates: Requirements 8.2**

### Property 14: Knowledge Base Evidence Citation

*For any* clinical reason in a risk assessment, it should include at least one evidence source from the knowledge base.

**Validates: Requirements 12.2, 12.3**

## Error Handling

### Input Validation Errors

**Scenario**: Malformed FHIR R4 data
- **Detection**: FHIR schema validation fails
- **Response**: Return HTTP 400 with ValidationResult containing specific errors
- **Logging**: Log validation failure (without PHI) for monitoring

**Scenario**: Missing required patient data fields
- **Detection**: PatientData extraction fails due to missing labs, vitals, or notes
- **Response**: Return HTTP 400 with descriptive error message
- **Logging**: Log missing fields for monitoring

### Model Invocation Errors

**Scenario**: Bedrock API throttling
- **Detection**: ThrottlingException from Bedrock API
- **Response**: Retry with exponential backoff (max 3 retries)
- **Fallback**: If retries exhausted, return HTTP 503 with retry-after header
- **Logging**: Log throttling events for capacity planning

**Scenario**: Model timeout
- **Detection**: Bedrock API timeout (> 30 seconds)
- **Response**: Retry once, then fail with HTTP 504
- **Logging**: Log timeout for monitoring

**Scenario**: Model returns malformed JSON
- **Detection**: JSON parsing fails on model output
- **Response**: Retry with clarified prompt, then fail with HTTP 500
- **Logging**: Log malformed output for model monitoring

### Knowledge Base Errors

**Scenario**: Knowledge base unavailable
- **Detection**: OpenSearch connection failure
- **Response**: Proceed with analysis using only patient data (no external references)
- **Warning**: Set confidence_score penalty and add warning to reasons
- **Logging**: Log knowledge base failure for infrastructure monitoring

**Scenario**: No relevant references found
- **Detection**: Semantic search returns empty results
- **Response**: Proceed with analysis, note lack of external evidence in reasons
- **Warning**: May result in lower confidence_score
- **Logging**: Log query terms for knowledge base improvement

### Database Errors

**Scenario**: Audit log write failure
- **Detection**: PostgreSQL connection or write error
- **Response**: Return prediction to user but log error asynchronously
- **Retry**: Retry audit log write in background (max 5 attempts)
- **Alert**: Alert administrators if audit logging consistently fails
- **Logging**: Log database errors for infrastructure monitoring

**Scenario**: Consent check failure
- **Detection**: PostgreSQL connection error during opt-out check
- **Response**: Fail-safe: reject request with HTTP 503
- **Rationale**: Cannot verify consent, must not process data
- **Logging**: Log consent check failures for infrastructure monitoring

### Security Errors

**Scenario**: Patient opted out
- **Detection**: PatientConsentManager.check_consent() returns False
- **Response**: Return HTTP 403 with message "Patient has opted out"
- **Logging**: Log opt-out enforcement (patient_id only, no PHI)

**Scenario**: Invalid authentication
- **Detection**: IAM authentication fails
- **Response**: Return HTTP 401 with authentication challenge
- **Logging**: Log authentication failures for security monitoring

### Output Validation Errors

**Scenario**: Risk score out of bounds
- **Detection**: Parsed risk_score < 0.0 or > 1.0
- **Response**: Clamp to valid range [0.0, 1.0] and log warning
- **Logging**: Log out-of-bounds values for model monitoring

**Scenario**: Fewer than 3 reasons returned
- **Detection**: Parsed reasons array length < 3
- **Response**: Retry model invocation with explicit instruction for 3 reasons
- **Fallback**: If retry fails, pad with generic reason and set low confidence
- **Logging**: Log insufficient reasons for model monitoring

### General Error Handling Principles

1. **Never expose PHI in error messages**: All error responses scrubbed of patient data
2. **Fail-safe for consent**: If consent cannot be verified, reject request
3. **Graceful degradation**: System can operate with reduced functionality (e.g., no knowledge base)
4. **Comprehensive logging**: All errors logged (without PHI) for monitoring and debugging
5. **Retry with backoff**: Transient failures retried with exponential backoff
6. **User-friendly messages**: Error responses provide actionable guidance for API clients

## Testing Strategy

### Dual Testing Approach

The system will use both unit tests and property-based tests to ensure comprehensive coverage:

**Unit Tests**: Verify specific examples, edge cases, and error conditions
- Specific FHIR R4 resource examples
- Edge cases (empty data, boundary values)
- Error conditions (malformed input, API failures)
- Integration points between components
- Mock external dependencies (Bedrock, Knowledge Base, PostgreSQL)

**Property-Based Tests**: Verify universal properties across all inputs
- Generate random patient data and verify output constraints
- Test with many iterations (minimum 100) to catch edge cases
- Validate invariants that must hold for all valid inputs
- Complement unit tests by covering input space comprehensively

### Property-Based Testing Configuration

**Framework**: Use Hypothesis for Python property-based testing

**Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with comment referencing design property
- Tag format: `# Feature: clinical-intervention-agent, Property N: [property text]`

**Test Organization**:
- Each correctness property implemented by a single property-based test
- Property tests placed close to implementation (catch errors early)
- Property tests annotated with requirements clause numbers

### Test Coverage Requirements

**Component Tests**:
- FHIR API Handler: Unit tests for validation, extraction, formatting
- Clinical Intervention Agent: Property tests for output constraints, unit tests for workflow
- Knowledge Base Client: Unit tests with mocked OpenSearch responses
- Bedrock Model Client: Unit tests with mocked Bedrock API responses
- Audit Logger: Property tests for log completeness, unit tests for PHI scrubbing
- Patient Consent Manager: Unit tests for opt-out enforcement

**Integration Tests**:
- End-to-end flow with test FHIR data
- Knowledge base integration with test embeddings
- Audit log persistence and retrieval
- Error handling scenarios

**Security Tests**:
- PHI exclusion from logs (property test)
- Opt-out enforcement (property test)
- Input validation against malicious payloads (unit tests)

### Example Property Test

```python
from hypothesis import given, strategies as st
import hypothesis

# Feature: clinical-intervention-agent, Property 1: Risk Score Bounds
@given(patient_data=st.builds(PatientData, ...))
@hypothesis.settings(max_examples=100)
def test_risk_score_bounds(patient_data):
    """For any patient data input, risk_score should be between 0.0 and 1.0"""
    agent = ClinicalInterventionAgent()
    assessment = agent.analyze_patient(patient_data)
    assert 0.0 <= assessment.risk_score <= 1.0
```

### Testing Priorities

1. **Critical Path**: Output structure validation (Properties 1-4)
2. **Compliance**: Audit logging and PHI exclusion (Properties 5-7, 9)
3. **Security**: Opt-out enforcement (Property 8)
4. **Interoperability**: FHIR R4 compliance (Properties 10, 12, 13)
5. **Quality**: Knowledge base integration (Properties 11, 14)

### Mocking Strategy

**External Services to Mock**:
- Amazon Bedrock API: Mock with predefined responses for deterministic testing
- Knowledge Base (OpenSearch): Mock with test clinical references
- PostgreSQL: Use in-memory database or test database for integration tests
- S3: Mock with local filesystem or MinIO for testing

**Mocking Principles**:
- Mock external dependencies for unit tests
- Use real services for integration tests (test environment)
- Property tests use mocked services for speed and determinism
- Document mock behavior to match real service contracts

### Performance Testing

While not part of automated unit/property tests, the following performance characteristics should be validated manually:

- **Latency**: End-to-end analysis completes in < 5 seconds (p95)
- **Throughput**: System handles 10 concurrent requests without degradation
- **Model Invocation**: Bedrock API calls complete in < 3 seconds (p95)
- **Knowledge Base Query**: Semantic search completes in < 500ms (p95)

### Compliance Testing

**HIPAA Compliance Validation**:
- Manual audit of all logging statements to verify PHI exclusion
- Review of error messages to ensure no PHI exposure
- Verification of encryption configuration (infrastructure tests)
- Audit log completeness review

**FHIR R4 Compliance Validation**:
- Validation against official FHIR R4 schemas
- Testing with FHIR validator tools
- Integration testing with FHIR sandbox environments
