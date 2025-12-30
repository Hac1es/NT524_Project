# Setup Keycloak with OpenStack Keystone

## Part 1: Keycloak Server Setup

### 1. Update System and Install Java

```bash
sudo apt update
sudo apt install openjdk-21-jdk
```

### 2. Download and Extract Keycloak

```bash
wget https://github.com/keycloak/keycloak/releases/download/26.4.5/keycloak-26.4.5.zip
unzip keycloak-26.4.5.zip
cd keycloak-26.4.5
```

### 3. Set Environment Variables & Initialize Run

```bash
export KC_BOOTSTRAP_ADMIN_USERNAME=admin
export KC_BOOTSTRAP_ADMIN_PASSWORD=admin123
bin/kc.sh start-dev
```

### 4. Create Systemd Service File

Create the service file:

```bash
sudo nano /etc/systemd/system/keycloak.service
Add the following content:

ini
[Unit]
Description=Keycloak Application
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/keycloak-26.4.5
Environment=JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
Environment=PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/home/ubuntu/keycloak-26.4.5/bin/kc.sh start \
          --http-enabled=true \
          --hostname-strict=false \
          --proxy-headers=xforwarded \
          --http-port=8088 \
          --log=file --log-file=/var/log/keycloak/keycloak.log \
          --log-level=debug
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### 5. Set Permissions and Enable Service

```bash
sudo chmod 644 /etc/systemd/system/keycloak.service
sudo systemctl daemon-reload
sudo systemctl start keycloak
sudo systemctl enable keycloak
```

### 6. Verify Installation

Check if Keycloak is running:

```bash
sudo systemctl status keycloak
```

You should now be able to access Keycloak admin console at:

```text
http://your-server-ip:8088
Username: admin
Password: admin123
```

---

## Part 2: Create Realm and Client for OpenStack Keystone

### 1. Create a New Realm

1. Access Keycloak Admin Console at `http://your-server-ip:8088`
2. Login with admin credentials (admin/admin123)
3. In the top-left dropdown, click on the current realm name
4. Click **"Create realm"**
5. Enter realm name: `Cloud`
6. Click **"Create"**

### 2. Create Client for Keystone Federation

1. Navigate to **Clients** in the left sidebar
2. Click **"Create"** button
3. Configure the client as follows:

**Client Settings:**

- Client ID: `keystone`
- Name: `keystone`
- Description: `Keystone Federation`
- Client Protocol: `openid-connect`
- Root URL: `https://192.168.1.254:5000/v3`

### 3. Configure Client Details

In the client settings page, configure the following:

**Settings Tab:**

- Access Type: `confidential`
- Standard Flow: `Enabled`
- Implicit Flow: `Enabled`
- Direct Access Grants: `Enabled`

- Valid Redirect URIs:

```text
https://192.168.1.254:5000/redirect_uri
https://192.168.1.254:5000/v3/auth/OS-FEDERATION/identity_providers/keycloak/protocols/openid/websso
https://192.168.1.254:5000/v3/*
```

- Web Origins: `https://192.168.1.254:5000`

- Admin URL: `https://192.168.1.254:5000`

### 4. Save Client Secret

1. Go to the **Credentials** tab of the keystone client
2. **Important:** Copy and save the **Client Secret** - this will be needed for Keystone configuration
3. The secret will look like: `abc123def456ghi789...`

### 5. Enable Client Authentication

Ensure the following are enabled:

- ✓ Client authentication
- ✓ Standard flow
- ✓ Implicit flow

Click **"Save"** to apply all changes.

---

## Part 3: OpenStack TLS setup

Enable TLS to avoid conflicts with Keycloak and secure communications.

### 1. Create Certificate Directories

```bash
sudo mkdir -p /etc/kolla/certificates/ca
```

### 2. Generate CA Certificate

```bash
# Generate CA private key
sudo openssl genrsa -out /etc/kolla/certificates/ca/ca.key 2048

# Generate CA certificate
sudo openssl req -x509 -new -nodes -key /etc/kolla/certificates/ca/ca.key \
  -sha256 -days 365 -out /etc/kolla/certificates/ca.crt \
  -subj "/C=VN/ST=HCM/L=HCM/O=Kolla/CN=CA"
```

### 3. Create SAN Configuration File

```bash
sudo nano /etc/kolla/certificates/san.cnf
```

Add the following content:

```ini
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = VN
ST = HCM
L = HCM
O = Kolla
CN = 192.168.1.254

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.1.254
DNS.1 = horizon.hippi.work
```

### 4. Generate OpenStack Server Certificates

```bash
# Generate private key
sudo openssl genrsa -out /etc/kolla/certificates/openstack.key 2048

# Generate certificate signing request
sudo openssl req -new -key /etc/kolla/certificates/openstack.key \
  -out /etc/kolla/certificates/openstack.csr \
  -subj "/C=VN/ST=HCM/L=HCM/O=Kolla/CN=horizon.hippi.work" \
  -config /etc/kolla/certificates/san.cnf

# Sign the certificate with CA
sudo openssl x509 -req -in /etc/kolla/certificates/openstack.csr \
  -CA /etc/kolla/certificates/ca.crt \
  -CAkey /etc/kolla/certificates/ca.key \
  -CAcreateserial \
  -out /etc/kolla/certificates/openstack.crt \
  -days 365 -sha256 \
  -extfile /etc/kolla/certificates/san.cnf \
  -extensions v3_req
```

### 5. Prepare HAProxy Certificates

```bash
# Combine certificate and key for HAProxy
sudo cat /etc/kolla/certificates/openstack.crt /etc/kolla/certificates/openstack.key | sudo tee /etc/kolla/certificates/haproxy.pem

# Copy for internal use
sudo cp /etc/kolla/certificates/haproxy.pem /etc/kolla/certificates/haproxy-internal.pem

# Set appropriate permissions
sudo chmod 666 /etc/kolla/certificates/haproxy.pem /etc/kolla/certificates/haproxy-internal.pem /etc/kolla/certificates/ca/ca.crt

# Install CA Certificate System-wide
sudo cp /etc/kolla/certificates/ca/ca.crt /usr/local/share/ca-certificates/kolla-customca-ca.crt
sudo update-ca-certificates
```

## Part 4: Configure OpenStack & Keystone

### 1. Edit the Kolla Ansible globals.yml file:

```bash
sudo nano /etc/kolla/globals.yml
```

Add/Modify the following sections:

```yaml
# Network settings
kolla_internal_vip_address: "192.168.1.254"
kolla_internal_fqdn: "{{ kolla_internal_vip_address }}"
kolla_enable_tls_external: "yes"
kolla_enable_tls_internal: "yes"
kolla_external_vip_address: "192.168.1.254"
kolla_external_fqdn: "{{ kolla_internal_vip_address }}"
kolla_external_fqdn_cert: "{{ kolla_certificates_dir }}/haproxy.pem"
kolla_internal_fqdn_cert: "{{ kolla_certificates_dir }}/haproxy-internal.pem"
kolla_tls_backend_cert: "{{ kolla_certificates_dir }}/haproxy.pem"
kolla_copy_ca_into_containers: "yes"
kolla_admin_openrc_cacert: "{{ kolla_certificates_dir }}/ca.crt"
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"

# OpenStack options
openstack_logging_debug: "True"
oidc_loglevel: "debug"

# Keystone Federation with Keycloak as Identity Provider
enable_keystone_federation: "yes"

keystone_identity_providers:
  - name: "keycloak"
    openstack_domain: "nt524"
    protocol: "openid"
    identifier: "https://sso.hippi.work/realms/cloud"
    public_name: "Authenticate via Keycloak"
    attribute_mapping: "keycloak_mapping"
    metadata_folder: "/etc/kolla/config/keystone/metadata"

keystone_identity_mappings:
  - name: "keycloak_mapping"
    file: "/etc/kolla/config/keystone/keycloak_mapping.json"

keystone_federation_oidc_additional_options:
  OIDCTokenBindingPolicy: disabled
```

### 2. Create Keystone Federation Directory Structure

sudo mkdir -p /etc/kolla/config/keystone/metadata
sudo mkdir -p /etc/kolla/config/keystone

Directory structure:

```text
/etc/kolla/config/keystone/
├── metadata/
│   ├── sso.hippi.work%2Frealms%2Fcloud.client
│   ├── sso.hippi.work%2Frealms%2Fcloud.conf
│   └── sso.hippi.work%2Frealms%2Fcloud.provider
└── keycloak_mapping.json
```

### 3. Create Configuration File

```bash
sudo nano /etc/kolla/config/keystone/metadata/sso.hippi.work%2Frealms%2Fcloud.client
```

Content:

```json
{
  "client_id": "keystone",
  "client_secret": "your-secret-from-keycloak"
}
```

Important: Replace your-secret-from-keycloak with the actual client secret from Keycloak (Part 2, Step 4).

```bash
sudo nano /etc/kolla/config/keystone/metadata/sso.hippi.work%2Frealms%2Fcloud.provider
```

Content:

```json
{
  "issuer": "https://sso.hippi.work/realms/cloud",
  "authorization_endpoint": "https://sso.hippi.work/realms/cloud/protocol/openid-connect/auth",
  "token_endpoint": "https://sso.hippi.work/realms/cloud/protocol/openid-connect/token",
  "userinfo_endpoint": "https://sso.hippi.work/realms/cloud/protocol/openid-connect/userinfo",
  "revocation_endpoint": "https://sso.hippi.work/realms/cloud/protocol/openid-connect/revoke",
  "jwks_uri": "https://sso.hippi.work/realms/cloud/protocol/openid-connect/certs",
  "response_types_supported": [
    "code",
    "token",
    "id_token",
    "code token",
    "code id_token",
    "id_token token",
    "code id_token token",
    "none"
  ],
  "subject_types_supported": ["public, pairwise"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "email", "profile"],
  "token_endpoint_auth_methods_supported": [
    "client_secret_post",
    "client_secret_basic"
  ],
  "claims_supported": [
    "aud",
    "email",
    "email_verified",
    "exp",
    "family_name",
    "given_name",
    "iat",
    "iss",
    "locale",
    "name",
    "picture",
    "sub"
  ],
  "code_challenge_methods_supported": ["plain", "S256"]
}
```

Note: This configuration can be copied from Keycloak's OpenID Connect discovery endpoint: `https://sso.hippi.work/realms/cloud/.well-known/openid-configuration`

Mapping file:

```bash
sudo nano /etc/kolla/config/keystone/keycloak_mapping.json
```

```json
[
  {
    "local": [
      {
        "user": {
          "name": "{0}",
          "email": "{1}",
          "domain": {
            "id": "c81f51f089a64cff943ed08978f560b0"
          },
          "type": "local"
        }
      }
    ],
    "remote": [
      {
        "type": "OIDC-iss",
        "any_one_of": ["https://sso.hippi.work/realms/cloud"]
      },
      {
        "type": "OIDC-preferred_username"
      },
      {
        "type": "OIDC-email"
      }
    ]
  }
]
```

### User Mapping Logic

Keystone performs the following user lookup process:

- **Extracts user attributes** from OIDC tokens:

  - `preferred_username` = unique username from identity provider
  - `email` = user's email address

- **Searches for existing user** in the `nt524` domain (ID: c81f51f089a64cff943ed08978f560b0) using:

  - Username matching `preferred_username`
  - Email address matching

- **Two possible outcomes**:
  - **User found**: If an existing user matches either username or email in the domain, authentication succeeds and user logs in
  - **User not found**: If no matching user exists, authentication fails with error (due to `local` mapping type restrictions)

### 5. Reconfig OpenStack:

```bash
cd /etc/kolla/ansible/inventory/
source ~/kolla-venv/bin/activate
sudo systemctl restart docker
sudo docker ps -q | xargs -r sudo docker stop
kolla-ansible reconfigure -i /etc/kolla/ansible/inventory/all-in-one
```

### 6. Edit /etc/kolla/admin-openrc.sh

```ini
export OS_AUTH_URL='https://192.168.1.254:35357/v3' -> http to https
export OS_CACERT=/etc/kolla/certificates/ca/ca.crt -> add this
```

## Part 5: Testing:

### 1. Create Keycloak user

Realm Cloud > User tungzeka/tung2005/nchinhtung@gmail.com

### 2. Create Keystone user

```bash
source ~/kolla-venv/bin/activate
cd /etc/kolla/ansible/inventory/
source /etc/kolla/admin-openrc.sh

# Create tungzeka/tung2005/nchinhtung@gmail.com
openstack project create Akali
openstack user create --project Akali --password tung2005 --email nchinhtung@gmail.com --domain nt524 tungzeka
openstack role add --project Akali --user tungzeka admin
```

#### Expected result

1. Access Horizon: Navigate to `https://192.168.1.254`
2. Select SSO: Click "Authenticate via Keycloak" on login page
3. Keycloak Redirect: Automatically redirected to Keycloak login page
   Enter Credentials: Use `tungzeka / tung2005`
4. Successful Authentication:

- Redirected back to Horizon dashboard
- Logged in as user `tungzeka` from domain `nt524`
- Full admin access to project `Akali`
