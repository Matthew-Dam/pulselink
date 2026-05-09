# PulseLink

**A Real-Time Consent-Based Friend Proximity & Coordination Platform**

> Enterprise-grade, privacy-first, AI-powered location synchronization for modern social coordination.

**Status**: Architecture & Planning Phase | **Version**: 1.0 Design | **Last Updated**: 2026-05-09

---

## 🎯 Product Vision

PulseLink solves the friction of coordinating with friends in real-time. Instead of endless texts "where are you?" or intrusive calls, users can:

1. **Scan for nearby friends** with a single tap
2. **Request consent-based location sharing** (temporary by default)
3. **Coordinate instantly** with directional guidance and smart meetup suggestions
4. **Stay safe** with anti-stalking protections and transparent consent logs

### Core Values

- 🔐 **Privacy-First**: Consent-based, temporary sharing, instant revocation
- 🤝 **Coordination Not Surveillance**: No silent tracking, no background data harvesting
- ⚡ **Instant & Intuitive**: Sub-second latency, effortless UX
- 🌍 **Scalable by Design**: Built for millions of concurrent users
- 🛡️ **Security Obsessed**: End-to-end encryption, zero-trust architecture, abuse detection

---

## 📋 Quick Navigation

### Core Documentation
- **[ARCHITECTURE.md](docs/ARCHITECTURE.md)** - Complete system design, microservices, infrastructure
- **[DATABASE_SCHEMA.md](docs/DATABASE_SCHEMA.md)** - PostgreSQL + PostGIS schema, indexing, optimization
- **[API_SPECIFICATION.md](docs/API_SPECIFICATION.md)** - REST & WebSocket endpoints
- **[SECURITY_MODEL.md](docs/SECURITY_MODEL.md)** - Encryption, authentication, abuse prevention
- **[SCALABILITY_STRATEGY.md](docs/SCALABILITY_STRATEGY.md)** - Horizontal scaling, sharding, caching
- **[DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md)** - Docker, Kubernetes, CI/CD
- **[AI_SYSTEM.md](docs/AI_SYSTEM.md)** - ML pipelines, recommendations, anomaly detection

---

## 🏗️ Technology Stack

| Layer | Technology | Justification |
|-------|------------|---------------|
| **Mobile** | Flutter (Dart) | Cross-platform, excellent real-time UX, strong location APIs |
| **Backend** | FastAPI (Python) | Native async, high performance, modern, easy to scale |
| **Real-Time** | WebSockets + Redis Pub/Sub | Low-latency, distributed messaging, excellent for location sync |
| **Database** | PostgreSQL 16 + PostGIS | ACID-compliant, best geospatial support, proven at scale |
| **Caching** | Redis Cluster | Ultra-fast session/location cache, pub/sub, rate limiting |
| **Infrastructure** | Kubernetes + Docker | Auto-scaling, self-healing, cloud-agnostic |
| **Observability** | Prometheus + Grafana + ELK | Real-time monitoring, alerting, log aggregation |
| **Cloud** | AWS/GCP (IaC via Terraform) | Global CDN, managed services, disaster recovery |

---

## 🚀 Quick Start

### Prerequisites
```bash
- Docker & Docker Compose 20.10+
- PostgreSQL 16+ (or use Docker)
- Redis 7+ (or use Docker)
- Python 3.11+
- Flutter 3.10+
- Node.js 20+ (for DevOps tools)
```

### Local Development (Docker Compose)

```bash
# Clone repository
git clone https://github.com/Matthew-Dam/pulselink.git
cd pulselink

# Start infrastructure (PostgreSQL, Redis, etc.)
docker-compose -f infrastructure/docker-compose.dev.yml up -d

# Backend setup
cd backend
python -m venv venv
source venv/bin/activate  # or: .\venv\Scripts\activate on Windows
pip install -r requirements.txt

# Initialize database
alembic upgrade head

# Start FastAPI server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# In new terminal, start Flutter app
cd ../mobile/pulselink_flutter
flutter pub get
flutter run
```

**Access Points:**
- API: `http://localhost:8000`
- API Docs: `http://localhost:8000/docs`
- WebSocket: `ws://localhost:8000/ws`
- Database: `localhost:5432`
- Redis: `localhost:6379`

---

## 📊 Architecture Overview

```
┌─────────────────────────────────────┐
│   Flutter Mobile (iOS/Android)      │
│  • Clean Architecture + BLoC        │
│  • GPS/BLE Sensor Fusion            │
│  • Offline-First                    │
│  • Local Encryption                 │
└────────────┬────────────────────────┘
             │ HTTPS/WSS (TLS 1.3)
             ▼
┌─────────────────────────────────────┐
│   Edge/CDN Layer                    │
│  • Cloudflare / AWS CloudFront      │
│  • Rate Limiting, DDoS Protection   │
│  • Geographic Routing               │
└────────────┬────────────────────────┘
             │
    ┌────────┴──────────┐
    ▼                   ▼
┌─────────────┐    ┌──────────────────┐
│ REST API LB │    │ WebSocket LB     │
│  NGINX/ALB  │    │ NGINX/ALB        │
└──────┬──────┘    └────────┬─────────┘
       │                    │
   ┌───┴────┐          ┌────┴─────┐
   ▼        ▼          ▼          ▼
 ┌────┐  ┌────┐   ┌──────┐   ┌──────┐
 │API1│  │API2│   │ WS-1 │   │ WS-2 │
 └────┘  └────┘   └──────┘   └──────┘
  FastAPI Services (Kubernetes Pods)
       │
    ┌──┴────────────────────────┐
    ▼                           ▼
┌─────────────────┐    ┌──────────────────┐
│ Redis Cluster   │    │ Message Bus      │
│ • Session Cache │    │ • NATS / Kafka   │
│ • Pub/Sub       │    │ • Event Streaming│
└─────────────────┘    └──────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ PostgreSQL 16 + PostGIS             │
│ • Sharded by Region/User ID         │
│ • Read Replicas                     │
│ • PITR (Point-in-Time Recovery)     │
└─────────────────────────────────────┘
```

---

## 🔐 Security Highlights

✅ **Multi-Factor Authentication**
- TOTP (Time-based One-Time Password)
- SMS OTP
- Biometric login support

✅ **Encryption**
- TLS 1.3 in transit
- AES-256 at rest
- End-to-End Encryption for location sessions

✅ **Privacy**
- Consent-based location sharing
- Temporary sessions (1-time, 30min, always)
- Instant revocation capability
- Location fuzzing option (±50m to ±5km)

✅ **Anti-Abuse**
- ML-based stalking detection
- Rate limiting
- Anomaly detection (impossible travel, teleportation)
- Behavioral scoring

✅ **Compliance**
- GDPR-ready (data export, erasure)
- CCPA compliance
- 7-year audit logging
- Consent audit trail

---

## 🎯 Core Features

### Phase 1: MVP (Weeks 1-12)
- [x] Email/Phone authentication
- [x] Basic location tracking (GPS)
- [x] Friend management
- [x] Request location with consent
- [x] Simple map view
- [x] WebSocket infrastructure
- [x] Database schema

### Phase 2: Beta (Weeks 13-24)
- [ ] BLE + GPS sensor fusion
- [ ] Advanced privacy controls (location fuzzing)
- [ ] AR navigation mode
- [ ] Offline resilience
- [ ] Performance optimization
- [ ] Launch on TestFlight/Google Play

### Phase 3: 1.0 Release (Weeks 25-36)
- [ ] Emergency features (SOS)
- [ ] Smart meetup suggestions (AI)
- [ ] Geofencing
- [ ] Friend groups
- [ ] Analytics dashboard
- [ ] Public launch

---

## 📈 Scalability Targets

- **DAU (Daily Active Users)**: 10M+
- **Concurrent WebSocket Connections**: 100K+ per node
- **Location Updates**: 50K/sec globally
- **API Latency**: <100ms P99
- **WebSocket Latency**: <50ms P99
- **Database Queries**: <10ms P99 (with caching)

---

## 🗂️ Repository Structure

```
pulselink/
├── docs/
│   ├── ARCHITECTURE.md              # System design, microservices
│   ├── DATABASE_SCHEMA.md           # PostgreSQL + PostGIS schema
│   ├── API_SPECIFICATION.md         # REST & WebSocket API
│   ├── SECURITY_MODEL.md            # Encryption, auth, abuse prevention
│   ├── SCALABILITY_STRATEGY.md      # Sharding, caching, load balancing
│   ├── DEPLOYMENT_GUIDE.md          # Docker, Kubernetes, CI/CD
│   └── AI_SYSTEM.md                 # ML pipelines and recommendations
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                  # FastAPI app entry point
│   │   ├── config.py                # Environment configuration
│   │   ├── auth/                    # Authentication service
│   │   ├── location/                # Location tracking service
│   │   ├── social/                  # Social graph service
│   │   ├── ws/                      # WebSocket gateway
│   │   ├── notifications/           # Push notification service
│   │   ├── security/                # Consent, abuse detection
│   │   ├── ai/                      # AI/ML models
│   │   └── shared/                  # Shared utilities, models
│   ├── migrations/                  # Alembic database migrations
│   ├── tests/                       # Unit & integration tests
│   ├── docker/
│   │   ├── Dockerfile
│   │   └── docker-compose.yml
│   ├── k8s/                         # Kubernetes manifests
│   ├── requirements.txt             # Python dependencies
│   └── README.md                    # Backend-specific docs
├── mobile/
│   └── pulselink_flutter/
│       ├── lib/
│       │   ├── main.dart
│       │   ├── config/              # Theme, routes, environment
│       │   ├── core/                # Error handling, network, platform
│       │   ├── data/                # Local datasources, models, repos
│       │   ├── domain/              # Entities, repositories, use cases
│       │   ├── presentation/        # BLoC, pages, widgets
│       │   └── shared/              # Constants, extensions, utils
│       ├── pubspec.yaml
│       ├── android/
│       ├── ios/
│       └── README.md                # Mobile-specific docs
├── infrastructure/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── vpc.tf
│   │   ├── kubernetes.tf
│   │   └── outputs.tf
│   ├── kubernetes/
│   │   ├── namespace.yaml
│   │   ├── postgres/
│   │   ├── redis/
│   │   ├── api/
│   │   ├── websocket/
│   │   └── monitoring/
│   ├── docker-compose.dev.yml
│   ├── docker-compose.prod.yml
│   └── README.md
├── scripts/
│   ├── setup.sh
│   ├── migrate-db.sh
│   ├── deploy-staging.sh
│   └── deploy-prod.sh
├── .github/
│   └── workflows/
│       ├── test.yml
│       ├── build.yml
│       └── deploy.yml
├── .env.example
├── .gitignore
├── LICENSE
└── README.md (this file)
```

---

## 🛠️ Development Workflow

### Local Development
```bash
# Create feature branch
git checkout -b feature/amazing-feature

# Make changes, commit
git add .
git commit -m "feat: add amazing feature"

# Push and create PR
git push origin feature/amazing-feature
```

### Testing
```bash
# Backend tests
cd backend
pytest tests/ -v --cov=app

# Flutter tests
cd mobile/pulselink_flutter
flutter test
```

### Deployment
```bash
# Staging
./scripts/deploy-staging.sh

# Production
./scripts/deploy-prod.sh
```

---

## 📚 Documentation

For detailed information, see:

1. **Architecture**: [ARCHITECTURE.md](docs/ARCHITECTURE.md)
   - System design
   - Microservices breakdown
   - Data flow
   - Technology decisions

2. **Database**: [DATABASE_SCHEMA.md](docs/DATABASE_SCHEMA.md)
   - PostgreSQL schema
   - PostGIS geospatial queries
   - Performance optimization
   - Indexing strategy

3. **API**: [API_SPECIFICATION.md](docs/API_SPECIFICATION.md)
   - REST endpoints
   - WebSocket events
   - Authentication flow
   - Error handling

4. **Security**: [SECURITY_MODEL.md](docs/SECURITY_MODEL.md)
   - Encryption strategy
   - MFA implementation
   - Consent model
   - Anti-abuse detection

5. **Scalability**: [SCALABILITY_STRATEGY.md](docs/SCALABILITY_STRATEGY.md)
   - Horizontal scaling
   - Database sharding
   - Caching layers
   - Load balancing

6. **Deployment**: [DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md)
   - Docker setup
   - Kubernetes manifests
   - CI/CD pipelines
   - Monitoring & alerting

7. **AI/ML**: [AI_SYSTEM.md](docs/AI_SYSTEM.md)
   - Meetup prediction
   - Anomaly detection
   - Behavioral scoring
   - Model training

---

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit changes (`git commit -m 'Add AmazingFeature'`)
4. Push to branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines.

---

## 📄 License

Proprietary. All rights reserved. © 2026 PulseLink. For more information, see [LICENSE](LICENSE).

---

## 📞 Support & Contact

- **Issues & Bugs**: [GitHub Issues](https://github.com/Matthew-Dam/pulselink/issues)
- **Documentation**: [/docs](docs/)
- **Email**: support@pulselink.app
- **Discord Community**: [Join Server](https://discord.gg/pulselink)

---

## 🙏 Acknowledgments

Built with:
- Flutter for beautiful cross-platform mobile
- FastAPI for high-performance async backend
- PostgreSQL for reliable, scalable data storage
- PostGIS for world-class geospatial capabilities
- Kubernetes for cloud-native orchestration

---

**Made with ❤️ for real-time social coordination.**
