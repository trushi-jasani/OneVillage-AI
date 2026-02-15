# OneVillage AI - Technical Design Document

## 1. System Architecture

### High-Level Architecture

**Layers:**
- **Presentation**: IVR Gateway (Twilio/Exotel), WhatsApp Business API, Admin Dashboard
- **Application**: Voice Processing, NLP, Recommendation Engine, User Management, Analytics
- **Data**: PostgreSQL (relational), MongoDB (logs), Redis (cache), Elasticsearch (search)
- **Integration**: Government APIs, Third-party services

**Architecture Pattern**: Microservices with message queues (RabbitMQ/Kafka)

## 2. Technology Stack

**Backend**: Python 3.11+ (FastAPI), Node.js (Express)  
**AI/ML**: Google Cloud Speech/Azure Speech, spaCy, BERT, TensorFlow, PyTorch  
**Database**: PostgreSQL 15+, MongoDB 6+, Redis 7+, Elasticsearch 8+  
**Voice**: Twilio/Exotel (IVR), WhatsApp Business API  
**Cloud**: AWS (EC2, S3, Lambda, RDS) / Azure  
**DevOps**: Docker, Kubernetes, GitHub Actions, Prometheus, Grafana  
**Frontend**: React.js, Material-UI

## 3. AI Components

### 3.1 Speech-to-Text
- **Service**: Google Cloud Speech-to-Text / Azure Speech / Whisper
- **Languages**: 10+ Indian languages with accent support
- **Features**: Real-time streaming, noise cancellation, code-mixing support

### 3.2 NLP Intent Detection
- **Model**: Fine-tuned multilingual BERT (mBERT)
- **Intents**: scheme_search, eligibility_check, application_help, job_search, healthcare_info, education_info
- **Entities**: Location, demographics, scheme category, time references
- **Accuracy**: 90%+ intent classification, 85%+ entity extraction

### 3.3 Recommendation Engine
- **Algorithm**: Hybrid (Rule-based + Content-based + Collaborative filtering)
- **Scoring**: `0.6×Eligibility + 0.25×Relevance + 0.10×Popularity + 0.05×Recency`
- **Features**: Real-time personalization, deadline-aware prioritization

### 3.4 Text-to-Speech
- **Service**: Google Cloud TTS / Azure TTS / Coqui TTS
- **Features**: Natural prosody, SSML support, speed control, emotion-aware synthesis
- **Optimization**: Pre-generated audio caching, adaptive bitrate

## 4. Database Design

### PostgreSQL Tables

**users**: user_id, phone_number, preferred_language, name, age, gender, occupation, state, district, village, income_category, family_size

**schemes**: scheme_id, scheme_name, scheme_code, category, description, eligibility_criteria (JSONB), benefits, documents_required (JSONB), application_process, state, is_central, is_active

**queries**: query_id, user_id, session_id, query_text, intent, entities (JSONB), response_text, schemes_recommended (JSONB), confidence_score, channel, language, duration_seconds

**user_scheme_interactions**: interaction_id, user_id, scheme_id, interaction_type

**operators**: operator_id, name, phone_number, role, assigned_villages (JSONB), state, district

**feedback**: feedback_id, user_id, query_id, rating, comments

### MongoDB Collections

**conversation_logs**: session_id, user_id, channel, messages[], created_at  
**analytics_events**: event_type, user_id, properties, timestamp

### Redis Cache

```
user:profile:{user_id} → TTL: 1 hour
scheme:details:{scheme_id} → TTL: 24 hours
session:{session_id} → TTL: 30 minutes
popular:schemes:{state} → TTL: 6 hours
```

## 5. API Design

**POST /api/v1/voice/process** - Process voice input, return transcription + response  
**GET /api/v1/schemes/search** - Search schemes by category, state, user  
**POST /api/v1/schemes/check-eligibility** - Check user eligibility for scheme  
**POST /api/v1/users/profile** - Create/update user profile  
**GET /api/v1/admin/analytics/usage** - Usage analytics  
**GET /api/v1/operator/dashboard** - Operator dashboard data

## 6. Voice Interaction Flow

### Phone Call (IVR)
```
User dials → Language selection → Welcome message → User speaks query 
→ STT → NLP → Recommendation → Response generation → TTS → Audio playback 
→ Follow-up prompt → Loop or end call
```

### WhatsApp Voice
```
User sends voice message → Webhook receives → Download audio → Identify user 
→ STT → NLP → Recommendation → TTS → Upload to CDN → Send voice response
```

**Session**: 30-min timeout, last 5 exchanges cached in Redis

**Error Handling**: Retry with fallback STT, clarification prompts, transfer to operator

## 7. Data Flow

```
User Device → IVR/WhatsApp → API Gateway → Voice Processing Service
→ [STT → NLP → Session Manager] → Business Logic [User/Scheme/Recommendation]
→ Response Generator → TTS → Delivery → User Device

Storage: PostgreSQL (structured), MongoDB (logs), Redis (cache), S3 (audio)
```

## 8. Security & Privacy

**Encryption**: TLS 1.3 (transit), AES-256 (rest), E2E for voice  
**Authentication**: Phone+OTP (users), JWT+RBAC (operators), MFA (admins)  
**Privacy**: Data minimization, 90-day voice retention, 1-year query logs, user deletion rights  
**Compliance**: GDPR principles, India's Digital Personal Data Protection Act  
**Security**: VPC, DDoS protection, rate limiting (100 req/min), audit logging

## 9. Scalability

**Horizontal Scaling**: Stateless services, auto-scaling (CPU>70% → scale up)  
**Load Balancing**: ALB with health checks, round-robin distribution  
**Database**: Read replicas, connection pooling, partitioning, sharding  
**Caching**: L1 (in-memory) → L2 (Redis) → L3 (CDN)  
**Capacity**: Current 10K concurrent users → 6-month target 50K concurrent users

## 10. Deployment Architecture (AWS)

**Compute**: EC2 (API servers), Lambda (webhooks), ECS/EKS (containers)  
**Storage**: S3 (audio), EBS (DB volumes), EFS (ML models)  
**Database**: RDS PostgreSQL (Multi-AZ), DocumentDB, ElastiCache Redis  
**Network**: VPC, ALB, CloudFront CDN, Route 53  
**Security**: IAM, KMS, WAF, Shield

**Topology**: Multi-AZ deployment with public/private subnets

**CI/CD**: GitHub Actions → Build/Test → Docker → ECR → Blue-Green deployment

**Monitoring**: Prometheus + Grafana (metrics), ELK Stack (logs), Jaeger (tracing), PagerDuty (alerts)

**DR**: Daily backups, cross-region replication, RTO: 1 hour, RPO: 5 minutes

## 11. Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| STT accuracy issues | High | Multi-provider fallback, fine-tuning, confidence scoring |
| High latency on 2G | High | Audio compression, CDN, caching, API optimization |
| Third-party API failures | High | Multi-provider strategy, circuit breaker, local fallback |
| Security breaches | Critical | E2E encryption, audits, penetration testing, incident response |
| Outdated content | Medium | Automated scraping, API integration, version control |
| Low user adoption | High | Awareness campaigns, simple onboarding, multilingual support |
| Scalability issues | High | Auto-scaling, load testing, queue-based processing |
| Funding constraints | High | Government partnerships, CSR funding, phased rollout |
| Inaccurate information | High | Verification workflow, official sources, user feedback |

---

