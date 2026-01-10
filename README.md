## Federation & SSO an toàn (SAML/OIDC) cho Multi‑Cloud

Một giải pháp Federation & Single Sign‑On an toàn cho môi trường hybrid/multi‑cloud (OpenStack + AWS).

### Mục Tiêu & Phạm Vi

- Thiết lập Keycloak làm IdP trung tâm cho OpenStack Keystone (OIDC) và AWS IAM Identity Center (SAML). Triển khai identity tập trung tại đây.
- Bảo mật toàn hệ thống (TLS/mTLS, signed tokens/assertions, TTL ngắn).
- Thu thập và chuẩn hoá log xác thực từ Keycloak/Keystone/AWS vào ELK.
- Rule ELK phát hiện hành vi bất thường (ví dụ “Impossible Travel”).
- Tự động hóa IR bằng Ansible Playbook.

### Kiến Trúc Tổng Quan

- IdP: Keycloak phát hành OIDC ID tokens cho Keystone và SAML assertions cho AWS (Home Token), đóng vai trò Global SIS/Token Exchange Service
- SP: OpenStack Keystone (OIDC) & AWS IAM Identity Center (SAML 2.0)
- Logging
  - Keycloak/Keystone logs qua Filebeat → Logstash → Elasticsearch → Kibana
  - CloudTrail từ S3 → Logstash.
- Incident Response
  - Alert qua Kibana Rules.
  - Ansible playbook revoke session.
