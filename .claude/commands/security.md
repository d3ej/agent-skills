---
description: Security audit — vulnerability detection, threat modeling, OWASP review, hardening recommendations
---

Invoke the agent-skills:security-and-hardening skill and activate the security-auditor agent perspective.

Audit the current changes or specified code for security vulnerabilities:

1. **Input handling** — Injection vectors (SQL, NoSQL, OS command, LDAP), XSS, file upload abuse, open redirects
2. **Auth & authorization** — Password hashing, session management, IDOR, rate limiting, JWT validation
3. **Data protection** — Secrets in code/logs, PII exposure, encryption in transit and at rest
4. **Infrastructure** — Security headers, CORS, dependency CVEs, error message leakage, least privilege
5. **Supply chain** — Dependency pinning, CI/CD pipeline security, container image provenance
6. **Cloud** — IAM over-permissions, public storage buckets, audit logging, secrets management

Classify every finding: Critical (block release) | High (fix before release) | Medium | Low | Info.

Every Critical and High finding must include a proof-of-concept exploitation scenario and a specific remediation with code example.
