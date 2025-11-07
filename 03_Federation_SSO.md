
# Capstone Project Outline  

## Đề tài 3: Triển khai Federation & SSO an toàn (SAML / OAuth2 / OpenID Connect) trên môi trường Multi‑Cloud

**Mô tả ngắn:** Thiết kế, triển khai và đánh giá một giải pháp Federation & Single Sign‑On an toàn cho môi trường hybrid/multi‑cloud (OpenStack + AWS/Azure), bao gồm Identity Provider (IdP) (ví dụ Keycloak), cấu hình SAML/OIDC, auditing, monitoring, và các playbook mitigations khi phát hiện sự cố.

---

### 1. Đặt vấn đề (Problem Statement)

- Trong môi trường multi‑cloud, quản lý danh tính & xác thực phân tán dẫn tới độ phức tạp cao: nhiều identity provider, user store, credential types.  
- Thiếu một hệ thống federation & SSO an toàn gây rủi ro: credential sprawl, misconfiguration trong trust relationships, replay attacks, token theft, hoặc privilege escalation thông qua role assumption.  
- Mục tiêu: xây dựng một giải pháp SSO/Federation tiêu chuẩn, an toàn, có audit/monitoring và khả năng phản ứng khi phát hiện sai lệch.

---

### 2. Mục tiêu dự án (Objectives)

- Thiết lập IdP (Keycloak hoặc tương đương) làm trung tâm federation cho OpenStack (Keystone) và public cloud (AWS/Azure).  
- Triển khai SAML 2.0 và OIDC/OAuth2 flows cho web console & API access.  
- Bảo đảm security hardening cho assertions/tokens: signing/encryption, audience restriction, short-lived tokens, session management, và MFA.  
- Thu thập events/auth logs và thiết kế rules để phát hiện token misuse / abnormal access.  
- Thiết kế playbooks tự động (hoặc bán tự động) cho mitigation (revoke session, disable client, rotate keys).

---

### 3. Kiến trúc & Công nghệ sử dụng (Architecture & Tech Stack)

- **Identity Provider (IdP):** Keycloak (open-source) — supports SAML & OIDC; hoặc Azure AD / Auth0 as alternatives.  
- **Service Providers (SP):** OpenStack Keystone (federation), AWS IAM SAML provider + Roles, Azure AD Enterprise App (SAML/OIDC).  
- **Protocol:** SAML 2.0 (SSO SAML assertions), OpenID Connect (OIDC) / OAuth2 for API access and modern SPs.  
- **Certificate / Key Management:** Vault / HSM / KMS (AWS KMS / Azure Key Vault) for signing keys.  
- **IaC & Automation:** Terraform (cloud resources) + Ansible (configuration).  
- **Monitoring & Logging:** CloudTrail (AWS), Keystone audit logs (OpenStack), Keycloak event logs, ELK stack (Filebeat → Logstash → Elasticsearch → Kibana) hoặc Splunk.  
- **Policy Engine & Governance:** Open Policy Agent (OPA)/Rego for dynamic access policies; optional HashiCorp Sentinel.  
- **Mitigation Automation:** Cloud Custodian, Ansible + scripts to revoke tokens, remove SAML provider etc.  
- **Testing Tools:** SAML Tracer (browser extension), SAMLtool.org, jwt.io, OpenID Connect debug endpoints.

---

### 4. Threat Model & Attack Scenarios (Use Cases)

- **T1 – Replay of SAML assertion:** attacker captures assertion and replays to SP.  
- **T2 – Unsigned or improperly validated assertion:** SP accepts unsigned or wrong-audience assertion.  
- **T3 – Stolen refresh token / OIDC token theft:** attacker uses refresh token to long‑living access.  
- **T4 – Misconfigured ACS / Audience / AssertionConsumerService:** allows assertion to be accepted by wrong SP.  
- **T5 – Compromised IdP admin account:** creation of malicious clients/roles/trusts.  
- **T6 – Clock skew exploitation:** tampering with NotBefore/NotOnOrAfter checks if clocks are unsynchronized.  
- **T7 – MFA bypass / weak MFA:** tokens issued without required second factor.

---

### 5. Thiết kế & Triển khai (Design & Implementation)

**A. Kiến trúc cao cấp (High-level Architecture)**  

- Keycloak (IdP) được triển khai trong OpenStack hoặc VM trên public cloud, cấu hình realm và client cho: OpenStack Keystone, AWS SAML provider, Azure enterprise app, Web apps.  
- Keystore: keys dùng để ký SAML assertions / OIDC ID tokens lưu trong Vault / KMS, không lưu plaintext trong repo.  
- Logging: Keycloak events → Filebeat → ELK; CloudTrail và Keystone logs → ELK.  
- Policy enforcement: OPA webhook tích hợp vào authentication flow (tuỳ chọn).

**B. Bước triển khai chi tiết**  

1. **Provision IdP: Keycloak setup**  
   - Cài Keycloak (container hoặc VM).  
   - Tạo Realm, Users, Groups, Role mappings.  
   - Tạo Clients cho: OpenStack (SAML SP), AWS (SAML), Azure app/OIDC client, internal web apps (OIDC).  
   - Thiết lập mappers để emit attributes cần thiết (email, roles, groups, eduPersonPrincipalName hoặc custom claim).  
   - Export SAML metadata XML, OIDC discovery endpoint.

2. **OpenStack Federation configuration**  
   - Bật federation trong Keystone: cấu hình identity provider và mapping rules.  
   - Map SAML attributes → Keystone groups/roles.  
   - Test SSO console login via Keycloak.

3. **AWS SAML Provider & Role setup**  
   - Tạo IAM SAML Provider (upload IdP metadata).  
   - Tạo IAM Roles with trust relationship to the SAML Provider; cấu hình RoleAttribute/SessionName mapping.  
   - Chỉ cấp quyền tối thiểu (principle of least privilege) cho assumed roles.

4. **Azure AD / OIDC integration (nếu dùng)**  
   - Đăng ký application / enterprise app, import metadata, hoặc sử dụng OIDC client IDs & secrets.  
   - Thiết lập claims mapping & conditional access (MFA rules).

5. **Key Management & Certificates**  
   - Tạo signing key (RSA/ECDSA), lưu private keys trong HashiCorp Vault hoặc KMS.  
   - Configure Keycloak to sign SAML assertions and sign/encrypt assertions when possible.  
   - Rollover plan: certificate rotation scripts + test.

6. **Security Hardening**  
   - Enforce signed assertions verification on SP.  
   - Audience restriction, ACS URL validation, Recipient check.  
   - Shorten token/session lifetime (e.g., session TTL 15–60 minutes).  
   - Enforce MFA for privileged roles (role binding to group requiring MFA).  
   - Strict certificate validation & TLS config (HSTS, strong ciphers).

7. **Monitoring & Detection rules**  
   - Collect events: Keycloak login events, failed assertions, CloudTrail AssumeRole events, Keystone federated login.  
   - Create SIEM rules for suspicious patterns (see Section 9).

8. **Mitigation Playbooks**  
   - Automated: revoke refresh tokens via Keycloak Admin API; disable SAML provider in AWS; revoke AWS console sessions via STS revoke (note AWS STS revoke limitations).  
   - Manual escalation: disable IdP client, rotate signing keys, force re-authentication for users.

---

### 6. Dữ liệu & Logs cần thu thập (Data & Logs)

- Keycloak events: login, token issuance, admin events (client creation), failed login.  
- OpenStack Keystone federation logs: mapping events, token issuance.  
- AWS CloudTrail: AssumeRole, ConsoleLogin, CreateSAMLProvider, CreateRole.  
- Azure Sign‑in logs (if used): token issuance, conditional access fail.  
- Network/web logs for ACS endpoints: request headers, IPs, user-agent.  
- Certificate/key rotation events (Vault logs).

**Normalization:** Map all authentication events to a common schema: `{timestamp, user_id, idp, sp, event_type, client_id, ip, geo, user_agent, outcome, attributes}`.

---

### 7. Threat Detection Rules & Heuristics (SIEM Rules)

- **Rule R1:** Multiple successful SSO logins for the same user from geographically distant IPs within short timespan → possible credential theft.  
- **Rule R2:** SAML assertion presented to SP with unexpected audience or wrong ACS URL → misconfiguration or attack.  
- **Rule R3:** Token replay: same assertion ID re-used → alert.  
- **Rule R4:** Unusual volume of AssumeRole calls to high-privilege roles → privilege escalation attempt.  
- **Rule R5:** New IdP client created or admin role assignment changed → audit & alert.  
- **Rule R6:** Failed signature validation events on SP side → possible tampering.  
- **Rule R7:** Refresh token usage from new IP/country → suspicious.

---

### 8. Testing & Validation (Test Plan)

**Functional tests:** SSO login flow for web console, API token exchange (OIDC), role assumption (AWS).  
**Security test cases:**  

- TC1: Replay attack — capture assertion then try to reuse after initial consumption → should be rejected.  
- TC2: Invalid audience/ACS — present assertion with altered audience → rejected.  
- TC3: Unsigned assertion acceptance — ensure SP rejects unsigned.  
- TC4: Token theft + refresh usage — refresh token misuse should be detectable/revocable.  
- TC5: Certificate rotation — ensure no downtime, and old assertions invalidated as appropriate.  
- TC6: MFA enforcement check — privileged role requires second factor.  
**Tools:** SAML Tracer, curl + samlRequest, jwt.io, automated integration tests (pytest + saml library mocks).

---

### 9. Đánh giá (Evaluation Metrics)

- **Security metrics:** number of successful attack simulations prevented; detection rate for token misuse; number of misconfigurations found (e.g., missing audience checks).  
- **Operational metrics:** avg auth latency (ms); success rate of SSO logins; false positives from SIEM rules (per week).  
- **Usability metrics:** user login success rate, average auth-time UX.  
- **Resilience metrics:** MTTR for identity incidents; time to rotate keys; downtime during rotation.

---

### 10. Mitigation & Incident Response (Playbooks)

- **Automated actions (scripted):** revoke user sessions, revoke refresh tokens (IdP API), disable SAML Provider in AWS (Cloud Custodian), disable OIDC client.  
- **SOC playbook:** triage steps (identify token, source IP, user actions), containment (disable account, block IP), eradication (revoke tokens, rotate keys), recovery (restore accounts, reset sessions), lessons learned.  
- **Rollback plan:** pre-stored rollback runbooks for accidental over‑remediation.

---

### 11. Rubric đánh giá (Suggested Grading Rubric)

- **Thiết lập môi trường & Identity federation (25%)**: IdP + SP đã cấu hình, SSO hoạt động cho OpenStack & AWS/Azure.  
- **Security hardening & key management (20%)**: signing/encryption, KMS/Vault integration, rotation plan.  
- **Detection & Monitoring (20%)**: logs thu thập, SIEM rules, dashboards.  
- **Testing & Attack simulations (20%)**: kịch bản tấn công thực thi và hệ thống phản ứng.  
- **Báo cáo & Deliverables (15%)**: repo, IaC, docs, demo video.

---

### 12. Milestones & Timeline (14 tuần đề xuất)

- Tuần 1–2: Nghiên cứu giao thức SAML/OIDC & design architecture.  
- Tuần 3–4: Cài đặt Keycloak, cấu hình realm & clients.  
- Tuần 5–6: Tích hợp OpenStack Keystone (federation) & test SSO.  
- Tuần 7–8: Tạo SAML provider & IAM roles trên AWS; (tuỳ chọn) Azure app.  
- Tuần 9: Key management + certificate rotation scripts.  
- Tuần 10: Logging pipeline (Filebeat → ELK) & basic dashboards.  
- Tuần 11: Viết SIEM rules & alerts.  
- Tuần 12: Simulate attacks, validate mitigations.  
- Tuần 13: Stress test & usability checks.  
- Tuần 14: Finalize report, slides, demo.

---

### 13. Deliverables (nộp cuối)

- Repo: Terraform/Ansible scripts, Keycloak realm export, AWS Terraform for SAML provider/roles.  
- Dashboards: Kibana/Splunk dashboards & SIEM rule definitions.  
- Playbooks: mitigation scripts (Cloud Custodian policies, Ansible playbooks).  
- Test suite: scripts for attack simulations and integration tests.  
- Video demo (5–10 mins), Technical report (15–25 pages), Slides (10–15 slides).

---

### 14. Ethical & Privacy Considerations

- Không sử dụng tài khoản/credential thật ngoài lab; sanitize logs (mask IP, email, ARNs) nếu chia sẻ.  
- Chỉ thực hiện tấn công trong môi trường lab; tránh gây ảnh hưởng tới bên thứ ba.  
- Quản lý private keys an toàn; không commit keys vào VCS.

---

### 15. Extensions & Advanced Ideas (Optional)

- Adaptive Conditional Access: dynamic risk scoring (device, geo, behavior) để áp dụng MFA tự động.  
- Identity Analytics: dùng ML để phát hiện anomalous login patterns.  
- SCIM-based provisioning/deprovisioning integration (automate user lifecycle).  
- WebAuthn / FIDO2 integration for passwordless & phishing-resistant auth.  
- Multi‑IdP federation: aggregator pattern to support multiple enterprise IdPs.

---
