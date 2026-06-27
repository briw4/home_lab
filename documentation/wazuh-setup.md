# Wazuh SIEM Setup - Single-Node All-in-One on Ubuntu

 **Environment:** All three components (Indexer, Manager, Dashboard) run on the same Ubuntu machine at `10.0.2.20` (VLAN 2).  
 This guide follows the official Wazuh step-by-step installation but documents the exact config changes and issues encountered in this homelab.


## Architecture

| Component       | Role                              | IP          | Port  |
|----------------|-----------------------------------|-------------|-------|
| Wazuh Indexer  | Stores alerts (OpenSearch-based)  | 10.0.2.20   | 9200  |
| Wazuh Manager  | Collects and analyzes agent data  | 10.0.2.20   | 1514  |
| Wazuh Dashboard| Web UI                            | 10.0.2.20   | 443   |

Certificate mapping:

| Component | Certificate file |
|-----------|-----------------|
| Indexer   | `node-1.pem`    |
| Manager   | `wazuh-1.pem`   |
| Dashboard | `dashboard.pem` |

---

## Step 1 - Generate Certificates (do this first)

Download the cert tool and template config:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```

Edit `config.yml` — set every IP to your server's IP:

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "10.0.2.20"
  server:
    - name: wazuh-1
      ip: "10.0.2.20"
  dashboard:
    - name: dashboard
      ip: "10.0.2.20"
```

 **Note on `node_type`:** Only needed if you run multiple Wazuh Manager nodes in a cluster (`master`/`worker`). With a single manager, leave it out.

Run the script to generate all certificates at once:

```bash
bash wazuh-certs-tool.sh -A
```

This creates a `wazuh-certificates/` folder with certs for all three components. Deploy them **before** starting any service.

---

## Step 2 - Wazuh Indexer

### Install

```bash
apt-get install debconf adduser procps gnupg apt-transport-https

curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && \
  chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | tee -a /etc/apt/sources.list.d/wazuh.list

apt-get update
apt-get -y install wazuh-indexer
```

### Configure `/etc/wazuh-indexer/opensearch.yml`

Key changes from the default:
- Changed `network.host` from `0.0.0.0` → `10.0.2.20`
- Added `discovery.type: single-node` (required for a single-node setup, otherwise the cluster won't start)

```yaml
network.host: "10.0.2.20"
node.name: "node-1"
cluster.initial_master_nodes:
  - "node-1"
discovery.type: single-node
node.max_local_storage_nodes: "3"
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer

plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.transport.resolve_hostname: false

plugins.security.authcz.admin_dn:
  - "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
  - "CN=node-1,OU=Wazuh,O=Wazuh,L=California,C=US"
plugins.security.restapi.roles_enabled:
  - "all_access"
  - "security_rest_api_access"
```

### Deploy certificates

```bash
mkdir -p /etc/wazuh-indexer/certs

cp ~/wazuh-certificates/node-1.pem /etc/wazuh-indexer/certs/indexer.pem
cp ~/wazuh-certificates/node-1-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-indexer/certs/
cp ~/wazuh-certificates/admin.pem /etc/wazuh-indexer/certs/
cp ~/wazuh-certificates/admin-key.pem /etc/wazuh-indexer/certs/

chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
chmod 750 /etc/wazuh-indexer/certs
chmod 640 /etc/wazuh-indexer/certs/*
chmod 750 /etc/wazuh-indexer
```

### Start and initialize

```bash
systemctl daemon-reload
systemctl enable wazuh-indexer
systemctl start wazuh-indexer

bash /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

### Verify

```bash
# Should return cluster info
curl -k -u admin https://10.0.2.20:9200

# Should show node-1 as the only node
curl -k -u admin https://10.0.2.20:9200/_cat/nodes?v
```

---

## Step 3 - Wazuh Manager + Filebeat

### Install

```bash
apt-get install gnupg apt-transport-https  # if not already done

apt-get -y install wazuh-manager
apt-get -y install filebeat
```

### Configure Filebeat

```bash
curl -so /etc/filebeat/filebeat.yml \
  https://packages.wazuh.com/4.14/tpl/wazuh/filebeat/filebeat.yml
```

In `/etc/filebeat/filebeat.yml`, set the indexer IP:

```yaml
output.elasticsearch:
  hosts: ["10.0.2.20:9200"]
  protocol: https
  username: ${username}
  password: ${password}
```

Store credentials in the Filebeat keystore (avoids plaintext passwords):

```bash
filebeat keystore create
echo admin | filebeat keystore add username --stdin --force
echo admin | filebeat keystore add password --stdin --force
```

Download the Wazuh alerts template:

```bash
curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz \
  | tar -xvz -C /usr/share/filebeat/module
```

### Deploy certificates for Filebeat

```bash
mkdir -p /etc/filebeat/certs

cp ~/wazuh-certificates/wazuh-1.pem /etc/filebeat/certs/filebeat.pem
cp ~/wazuh-certificates/wazuh-1-key.pem /etc/filebeat/certs/filebeat-key.pem
cp ~/wazuh-certificates/root-ca.pem /etc/filebeat/certs/

chmod 500 /etc/filebeat/certs
chmod 400 /etc/filebeat/certs/*
chown -R root:root /etc/filebeat/certs
```

### Configure the Wazuh Manager to talk to the Indexer

Store indexer credentials in the Wazuh keystore:

```bash
echo 'admin' | /var/ossec/bin/wazuh-keystore -f indexer -k username
echo 'admin' | /var/ossec/bin/wazuh-keystore -f indexer -k password
```

In `/var/ossec/etc/ossec.conf`, add/update the `<indexer>` block:

```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://10.0.2.20:9200</host>
  </hosts>
  <ssl>
    <certificate_authorities>
      <ca>/etc/filebeat/certs/root-ca.pem</ca>
    </certificate_authorities>
    <certificate>/etc/filebeat/certs/filebeat.pem</certificate>
    <key>/etc/filebeat/certs/filebeat-key.pem</key>
  </ssl>
</indexer>
```

### Start services

```bash
systemctl daemon-reload
systemctl enable wazuh-manager && systemctl start wazuh-manager
systemctl enable filebeat && systemctl start filebeat

systemctl status wazuh-manager
```

---

## Step 4 - Wazuh Dashboard

### Install

```bash
apt-get install debhelper tar curl libcap2-bin
apt-get -y install wazuh-dashboard
```

### Configure `/etc/wazuh-dashboard/opensearch_dashboards.yml`

Key change: `server.host` from `0.0.0.0` → `10.0.2.20`

```yaml
server.host: 10.0.2.20
server.port: 443
opensearch.hosts: https://localhost:9200
opensearch.ssl.verificationMode: certificate
opensearch.requestHeadersAllowlist: ["securitytenant","Authorization"]
opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
server.ssl.enabled: true
server.ssl.key: "/etc/wazuh-dashboard/certs/dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/dashboard.pem"
opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]
uiSettings.overrides.defaultRoute: /app/wz-home
opensearch_security.cookie.ttl: 900000
opensearch_security.session.ttl: 900000
opensearch_security.session.keepalive: true
```

### Deploy certificates

```bash
mkdir -p /etc/wazuh-dashboard/certs

cp ~/wazuh-certificates/dashboard.pem /etc/wazuh-dashboard/certs/
cp ~/wazuh-certificates/dashboard-key.pem /etc/wazuh-dashboard/certs/
cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-dashboard/certs/

chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
chmod 750 /etc/wazuh-dashboard/certs
chmod 640 /etc/wazuh-dashboard/certs/*
chmod 750 /etc/wazuh-dashboard
```

### Point the Dashboard at the Wazuh Manager

In `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml`, replace `https://localhost` with the actual manager IP:

```yaml
hosts:
  - default:
      url: https://10.0.2.20
      port: 55000
      username: wazuh-wui
      password: wazuh-wui
      run_as: true
```

### Start

```bash
systemctl daemon-reload
systemctl enable wazuh-dashboard
systemctl start wazuh-dashboard
```

Access the UI at: `https://10.0.2.20` — default credentials are `admin` / `admin`.

---

## Step 5 - Install Wazuh Agents

### Windows (tested on Windows 10 and AD server)

Download the [Windows MSI installer](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi), then from an **Administrator CMD**:

```cmd
msiexec.exe /i wazuh-agent-4.14.5-1.msi /q WAZUH_MANAGER="10.0.2.20"
```
```cmd
NET START WazuhSvc
```

#### ⚠️ Issue encountered on the AD server

After installation the agent failed to start, with no error shown. Found the cause in `C:\Program Files (x86)\ossec-agent\ossec.conf`:

```xml
<!-- Wrong — default address was 0.0.0.0 -->
<client>
  <server>
    <address>0.0.0.0</address>
  </server>
</client>
```

Fix: replace `0.0.0.0` with the actual manager IP:

```xml
<client>
  <server>
    <address>10.0.2.20</address>
  </server>
</client>
```

Then restart:

```cmd
net start wazuh
```

---

## References

- [Official Wazuh step-by-step install guide](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html)
- [Wazuh agent installation on Windows](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)
- [Certificate deployment guide](https://documentation.wazuh.com/current/user-manual/wazuh-dashboard/certificates.html)
