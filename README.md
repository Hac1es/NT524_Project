## Secure Federation & SSO for Multi/Hybrid Cloud

A secure Federation and Single Sign-On solution for hybrid and multi-cloud environments.

### Issues

Fragmented identity management across **public** and **private** cloud leads to credential sprawl, misconfigurations, and fragmented auditing (**The MxN problem**)
⇒ A centralized SSO solution is required to unify identity management across hybrid cloud environments.

### Objectives & Scope

- Centralized identity across private & public cloud
- Hardened authentication & session security
- Unified logging & monitoring
- Anom﻿aly detection, threat visibility & semi-automated response

### Solution Architecture

<img src="SolutionArchitecture.png" width="90%">

A centralized identity fabric enables secure SSO across hybrid cloud. Global STS authenticates users, exchanges tokens, and enforces policy decisions, while platform IAM services map access rights. All activities are centrally monitored for security visibility.

### Implement Architecture

<img src="Architecture/Trien_khai_thuc_te.png" width="50%">

Due to time constraints, the system was implemented in a simplified form: Keycloak serves as Global STS, PDP, and token exchange service. Users authenticate once to obtain a Home Token for AWS and OpenStack access. Logs are centralized in ELK, with incident response automated via Ansible.

Although simplified, the architecture preserves the security model via centralized identity, token federation, and semi-automated monitoring & response.

### Scenarios

- Functionality Test: [SSO login via Keycloak to Horizon and seamless access to AWS Console; all auth logs centralized in ELK.](https://drive.google.com/file/d/1TjOLxGEJ8fo7UI2phOakoOcEAKyJW2Jy/view?usp=drive_link)
- MFA Enforcement: [Keycloak enforces OTP-based MFA with mandatory authenticator verification.](https://drive.google.com/file/d/1DsomiBwTJjKQyOZsDoJzArPhzlRQzdve/view?usp=drive_link)
- Unsigned Token: [Tampered or unsigned OIDC tokens are rejected, preventing unauthorized access.](https://drive.google.com/file/d/1xPBricIsjhagQGyEuVz8kw95FnPO1OG/view?usp=sharing)
- Impossible Travel Alert: [Geo-anomalous logins trigger Kibana alerts and semi-automatic session revocation with Ansible.](https://drive.google.com/file/d/1pWakjtDTkG07PD50c7MPgToRzA8kF0x/view?usp=drive_link) 
- Replay Attack: [Replayed authentication requests are blocked and logged as LOGIN_ERROR.](https://drive.google.com/file/d/1vVZVkNL7cYqv7rAnJDKhQS3Po2zYEWhs/view?usp=sharing)

### Future Impovements

- Vault-based secret management & key rotation
- Ingress layer for unified access & routing with Traefik & ModSecurity
- OPA-driven centralized policy enforcement for JiT access provisioning and ABAC
- GitOps workflow as single source of truth
