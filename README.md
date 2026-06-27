
# SSL certs generation guide


## Step 1 — CA workstation setup

This is done on a dedicated offline/air-gapped machine — never on any of the 8 VMs.

### 1.1 Create the correct directory structure

```
# Root CA directories
mkdir -p /opt/mtls-pki/ca/{certs,crl,newcerts,private}
chmod 700 /opt/mtls-pki/ca/private

# Intermediate CA directories
mkdir -p /opt/mtls-pki/intermediate/{certs,crl,newcerts,private}
chmod 700 /opt/mtls-pki/intermediate/private

# Issued node certs (one subdir per VM)
mkdir -p /opt/mtls-pki/issued/{vm1-filebeat,vm2-logstash1,vm3-logstash2,vm4-logstash3,vm5-haproxy,vm6-rsyslog1,vm7-rsyslog2,vm8-rsyslog3}

# Scripts
mkdir -p /opt/mtls-pki/scripts

# Root CA database
touch /opt/mtls-pki/ca/index.txt
echo 1000 > /opt/mtls-pki/ca/serial
echo 1000 > /opt/mtls-pki/ca/crlnumber

# Intermediate CA database
touch /opt/mtls-pki/intermediate/index.txt
echo 2000 > /opt/mtls-pki/intermediate/serial
echo 2000 > /opt/mtls-pki/intermediate/crlnumber

# Verify
find /opt/mtls-pki -type d | sort
```
### Step 2 — Root CA config
Save as /opt/mtls-pki/ca/openssl-root.cnf:

```
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /opt/mtls-pki/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 3650
preserve          = no
policy            = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 4096
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca
prompt              = no

[ req_distinguished_name ]
countryName             = IN
stateOrProvinceName     = Telangana
localityName            = Hyderabad
organizationName        = MyOrg
organizationalUnitName  = IT Security
commonName              = MyOrg Root CA

[ v3_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true, pathlen:1
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
authorityKeyIdentifier = keyid:always
```
### Step 3 — Generate Root CA (ECDSA P-256)

```
# Generate Root CA private key — ECDSA P-256
openssl ecparam \
  -name prime256v1 \
  -genkey \
  -noout \
  -out /opt/mtls-pki/ca/private/ca.key.pem

# Encrypt the key at rest (you will be prompted for a passphrase — keep it safe)
openssl pkcs8 \
  -topk8 \
  -v2 aes-256-cbc \
  -in  /opt/mtls-pki/ca/private/ca.key.pem \
  -out /opt/mtls-pki/ca/private/ca.key.enc.pem

mv /opt/mtls-pki/ca/private/ca.key.enc.pem /opt/mtls-pki/ca/private/ca.key.pem
chmod 400 /opt/mtls-pki/ca/private/ca.key.pem

# Self-sign the Root CA cert (10 years)
openssl req \
  -config  /opt/mtls-pki/ca/openssl-root.cnf \
  -key     /opt/mtls-pki/ca/private/ca.key.pem \
  -new -x509 \
  -days 3650 \
  -sha256 \
  -extensions v3_ca \
  -out /opt/mtls-pki/ca/certs/ca.cert.pem

chmod 444 /opt/mtls-pki/ca/certs/ca.cert.pem

# Verify — you must see CA:TRUE and pathlen:1
openssl x509 -noout -text -in /opt/mtls-pki/ca/certs/ca.cert.pem | \
  grep -A4 "Basic Constraints"

```
Expected output:
```
X509v3 Basic Constraints: critical
    CA:TRUE, pathlen:1
```
### Step 4 — Intermediate CA config

Save as /opt/mtls-pki/intermediate/openssl-intermediate.cnf:

```
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /opt/mtls-pki/intermediate
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/intermediate.key.pem
certificate       = $dir/certs/intermediate.cert.pem
crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 15
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 825
preserve          = no
policy            = policy_loose

[ policy_loose ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 4096
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
prompt              = no

[ req_distinguished_name ]
countryName             = IN
stateOrProvinceName     = Telangana
localityName            = Hyderabad
organizationName        = MyOrg
organizationalUnitName  = IT Security
commonName              = MyOrg Intermediate CA

[ server_cert ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth

[ client_cert ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = clientAuth

[ server_client_cert ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth, clientAuth

[ crl_ext ]
authorityKeyIdentifier = keyid:always
```
### Step 5 — Generate and sign the Intermediate CA

```
# Generate Intermediate CA key — ECDSA P-256
openssl ecparam \
  -name prime256v1 \
  -genkey \
  -noout \
  -out /opt/mtls-pki/intermediate/private/intermediate.key.pem

# Encrypt it
openssl pkcs8 \
  -topk8 -v2 aes-256-cbc \
  -in  /opt/mtls-pki/intermediate/private/intermediate.key.pem \
  -out /opt/mtls-pki/intermediate/private/intermediate.key.enc.pem

mv /opt/mtls-pki/intermediate/private/intermediate.key.enc.pem \
   /opt/mtls-pki/intermediate/private/intermediate.key.pem
chmod 400 /opt/mtls-pki/intermediate/private/intermediate.key.pem

# Generate CSR for the Intermediate CA
openssl req \
  -config /opt/mtls-pki/intermediate/openssl-intermediate.cnf \
  -key    /opt/mtls-pki/intermediate/private/intermediate.key.pem \
  -new -sha256 \
  -out /opt/mtls-pki/intermediate/certs/intermediate.csr.pem

# Root CA signs the Intermediate CA cert (5 years, pathlen:0)
openssl ca \
  -config     /opt/mtls-pki/ca/openssl-root.cnf \
  -extensions v3_intermediate_ca \
  -days       1825 \
  -notext -md sha256 \
  -in  /opt/mtls-pki/intermediate/certs/intermediate.csr.pem \
  -out /opt/mtls-pki/intermediate/certs/intermediate.cert.pem

chmod 444 /opt/mtls-pki/intermediate/certs/intermediate.cert.pem

# Build the CA chain file (intermediate + root, ordered)
cat /opt/mtls-pki/intermediate/certs/intermediate.cert.pem \
    /opt/mtls-pki/ca/certs/ca.cert.pem \
  > /opt/mtls-pki/intermediate/certs/ca-chain.cert.pem
chmod 444 /opt/mtls-pki/intermediate/certs/ca-chain.cert.pem

# Verify the intermediate against root — must say OK
openssl verify \
  -CAfile /opt/mtls-pki/ca/certs/ca.cert.pem \
  /opt/mtls-pki/intermediate/certs/intermediate.cert.pem

# Verify pathlen:0
openssl x509 -noout -text \
  -in /opt/mtls-pki/intermediate/certs/intermediate.cert.pem | \
  grep -A4 "Basic Constraints"
```
Expected:
```
X509v3 Basic Constraints: critical
    CA:TRUE, pathlen:0
```
### Step 6 — Generate CRL for both CAs

```
# Root CA CRL
openssl ca \
  -config /opt/mtls-pki/ca/openssl-root.cnf \
  -gencrl \
  -out /opt/mtls-pki/ca/crl/ca.crl.pem

# Intermediate CA CRL
openssl ca \
  -config /opt/mtls-pki/intermediate/openssl-intermediate.cnf \
  -gencrl \
  -out /opt/mtls-pki/intermediate/crl/intermediate.crl.pem

# Combine for distribution (HAProxy and Logstash consume this)
cat /opt/mtls-pki/intermediate/crl/intermediate.crl.pem \
    /opt/mtls-pki/ca/crl/ca.crl.pem \
  > /opt/mtls-pki/intermediate/crl/ca-chain.crl.pem

# Verify both CRLs
openssl crl -noout -text -in /opt/mtls-pki/ca/crl/ca.crl.pem | head -10
openssl crl -noout -text -in /opt/mtls-pki/intermediate/crl/intermediate.crl.pem | head -10
```
### Step 7 — Node certificate generation script
Save as /opt/mtls-pki/scripts/gen-cert.sh:
```
#!/bin/bash
# Usage: ./gen-cert.sh <hostname> <ip> <role>
# role: server | client | server_client
set -euo pipefail

HOSTNAME="$1"
IP="$2"
ROLE="$3"
DAYS=825     # Max enforced by Chrome/Apple — never exceed

INTDIR="/opt/mtls-pki/intermediate"
OUTDIR="/opt/mtls-pki/issued/${HOSTNAME}"
mkdir -p "${OUTDIR}"

echo ">>> Generating ECDSA P-256 key for ${HOSTNAME}"
openssl ecparam \
  -name prime256v1 \
  -genkey -noout \
  -out "${OUTDIR}/${HOSTNAME}.key.pem"
chmod 400 "${OUTDIR}/${HOSTNAME}.key.pem"

# Build SAN extension file — this is what fixes the "san section" error
# We use -extfile with openssl x509, NOT openssl ca -extensions
EXT_FILE="$(mktemp /tmp/san-XXXXXX.cnf)"
cat > "${EXT_FILE}" <<EXTEOF
[ req ]
distinguished_name = req_distinguished_name
prompt = no

[ req_distinguished_name ]
C  = IN
ST = Telangana
L  = Hyderabad
O  = MyOrg
CN = ${HOSTNAME}

[ v3_req ]
basicConstraints       = critical, CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
subjectAltName         = @alt_names
keyUsage               = critical, digitalSignature, keyEncipherment
EXTEOF

case "${ROLE}" in
  server)        echo "extendedKeyUsage = serverAuth"             >> "${EXT_FILE}" ;;
  client)        echo "extendedKeyUsage = clientAuth"             >> "${EXT_FILE}" ;;
  server_client) echo "extendedKeyUsage = serverAuth, clientAuth" >> "${EXT_FILE}" ;;
  *) echo "Unknown role: ${ROLE}. Use server|client|server_client"; exit 1 ;;
esac

cat >> "${EXT_FILE}" <<ALTEOF

[ alt_names ]
DNS.1 = ${HOSTNAME}
IP.1  = ${IP}
ALTEOF

echo ">>> Generating CSR"
openssl req \
  -new -sha256 \
  -key    "${OUTDIR}/${HOSTNAME}.key.pem" \
  -config "${EXT_FILE}" \
  -out    "${OUTDIR}/${HOSTNAME}.csr.pem"

echo ">>> Signing with Intermediate CA"
# Use openssl x509 -req + -extfile to avoid the [san] section issue entirely
openssl x509 -req \
  -in      "${OUTDIR}/${HOSTNAME}.csr.pem" \
  -CA      "${INTDIR}/certs/intermediate.cert.pem" \
  -CAkey   "${INTDIR}/private/intermediate.key.pem" \
  -CAserial "${INTDIR}/serial" \
  -days    "${DAYS}" \
  -sha256 \
  -extfile "${EXT_FILE}" \
  -extensions v3_req \
  -out     "${OUTDIR}/${HOSTNAME}.cert.pem"

chmod 444 "${OUTDIR}/${HOSTNAME}.cert.pem"

# Full chain: node cert + intermediate cert + root cert
cat "${OUTDIR}/${HOSTNAME}.cert.pem" \
    "${INTDIR}/certs/intermediate.cert.pem" \
    "/opt/mtls-pki/ca/certs/ca.cert.pem" \
  > "${OUTDIR}/${HOSTNAME}.chain.pem"

# Verify the signed cert against the full chain
openssl verify \
  -CAfile "${INTDIR}/certs/ca-chain.cert.pem" \
  "${OUTDIR}/${HOSTNAME}.cert.pem"

# Print SANs and dates for confirmation
openssl x509 -noout -subject -issuer -dates \
  -ext subjectAltName,extendedKeyUsage \
  -in "${OUTDIR}/${HOSTNAME}.cert.pem"

rm -f "${EXT_FILE}"
echo ">>> Done: ${OUTDIR}/"
ls -lh "${OUTDIR}/"
```
Give the permissions to it

```
chmod +x /opt/mtls-pki/scripts/gen-cert.sh
```
### Step 8 — Generate all 8 node certificates

```
cd /opt/mtls-pki/scripts

# VM1 — Filebeat: connects OUT to HAProxy → client only
./gen-cert.sh vm1-filebeat   10.0.1.11  client

# VM2 — Logstash1: accepts connections (server) + connects to rsyslog (client)
./gen-cert.sh vm2-logstash1  10.0.1.12  server_client

# VM3 — Logstash2
./gen-cert.sh vm3-logstash2  10.0.1.13  server_client

# VM4 — Logstash3
./gen-cert.sh vm4-logstash3  10.0.1.14  server_client

# VM5 — HAProxy: receives from Filebeat (server) + connects to Logstash (client)
./gen-cert.sh vm5-haproxy    10.0.1.15  server_client

# VM6 — rsyslog1: Logstash connects TO it → server only
./gen-cert.sh vm6-rsyslog1   10.0.1.16  server

# VM7 — rsyslog2
./gen-cert.sh vm7-rsyslog2   10.0.1.17  server

# VM8 — rsyslog3
./gen-cert.sh vm8-rsyslog3   10.0.1.18  server
```
Bulk verification — all 8 certs must say OK

```
CHAIN="/opt/mtls-pki/intermediate/certs/ca-chain.cert.pem"
for vm in vm1-filebeat vm2-logstash1 vm3-logstash2 vm4-logstash3 \
          vm5-haproxy vm6-rsyslog1 vm7-rsyslog2 vm8-rsyslog3; do
  echo -n "${vm}: "
  openssl verify -CAfile "${CHAIN}" \
    "/opt/mtls-pki/issued/${vm}/${vm}.cert.pem"
done
```
### Step 10 — Distribute certificates securely
Distribute the certificates securely
