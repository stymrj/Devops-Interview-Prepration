# Zero Trust & mTLS Basics — Hands-On Lab

Ten exercises to build real muscle memory around mTLS, certificates, and zero-trust enforcement. You'll need a terminal with `openssl` and optionally `kubectl` (minikind or minikube) for the mesh exercises.

---

## Exercise 1: Inspect a Real Certificate

**Goal:** Learn to read a certificate's identity, issuer, and expiry.

**Commands:**

```bash
echo | openssl s_client -connect github.com:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

**Expected output:** The subject (CN or SAN), the issuing CA, and `notBefore`/`notAfter` dates.

**Why it matters:** Every "certificate verify failed" debugging session starts here — reading the cert before guessing.

---

## Exercise 2: Create Your Own CA and Sign a Server Cert

**Goal:** Build a private CA from scratch and see how trust chains work.

**Commands:**

```bash
# Root CA
openssl req -x509 -newkey rsa:2048 -keyout ca.key -out ca.crt \
  -days 365 -nodes -subj "/CN=lab-ca"

# Server key + CSR + signed cert
openssl req -newkey rsa:2048 -keyout server.key -out server.csr \
  -nodes -subj "/CN=payments.internal"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 30
```

**Expected output:** `ca.crt`, `server.crt`, `server.key` — a 30-day server cert signed by your CA.

**Why it matters:** Internal CAs work exactly like this; the only difference in production is automation and HSM-backed keys.

---

## Exercise 3: Verify the Chain (and Watch It Fail)

**Goal:** See chain verification succeed and fail depending on trust.

**Commands:**

```bash
openssl verify -CAfile ca.crt server.crt          # should say OK
openssl verify server.crt                         # should FAIL
```

**Expected output:** `server.crt: OK` for the first, an "unable to get local issuer certificate" error for the second.

**Why it matters:** This is the mechanics behind "works on my machine but not in the pod" — the trust store differs, not the cert.

---

## Exercise 4: Run an mTLS-Protected Server with curl

**Goal:** Experience the handshake from both sides with a real server.

**Commands:**

```bash
# Client cert signed by same CA
openssl req -newkey rsa:2048 -keyout client.key -out client.csr \
  -nodes -subj "/CN=checkout-service"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt -days 30

# Start an HTTPS server that REQUIRES client certs
openssl s_server -accept 8443 -cert server.crt -key server.key \
  -CAfile ca.crt -Verify 1 -www &

# Try without a client cert (fails), then with one (works)
curl -sk https://localhost:8443/ ; echo "---"
curl -sk --cert client.crt --key client.key https://localhost:8443/
```

**Expected output:** First curl errors with an SSL handshake failure; second returns the s_server status page.

**Why it matters:** That failure *is* mTLS — the server refused an unauthenticated client before any data moved.

---

## Exercise 5: Read an mTLS Handshake in Wireshark/tshark

**Goal:** Spot the `CertificateRequest` message that makes mTLS mutual.

**Commands:**

```bash
tshark -i lo -f "tcp port 8443" -Y "tls.handshake.type" \
  -T fields -e tls.handshake.type -e tls.handshake.extensions_server_name
```

**Expected output:** Handshake types `1` (ClientHello), `2` (ServerHello), `11` (Certificate), `13` (CertificateRequest), `15` (CertificateVerify)...

**Why it matters:** Interviewers love "walk me through the handshake" — seeing the actual packets makes it stick.

---

## Exercise 6: Simulate Cert Expiry Chaos

**Goal:** Reproduce the classic rolling-outage pattern of expired certs.

**Commands:**

```bash
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server-expired.crt -days 0
openssl x509 -in server-expired.crt -noout -dates
curl -sk --cert client.crt --key client.key https://localhost:8443/ || true
```

**Expected output:** A cert whose `notAfter` is already past, and handshake failures the moment a fresh connection is attempted.

**Why it matters:** Live connections survive expiry; new ones don't. That's the rolling-outage signature.

---

## Exercise 7: Enforce mTLS in Istio with STRICT Mode (minikube/kind)

**Goal:** Turn on mesh-wide mTLS declaratively and watch plaintext fail.

**Commands:**

```bash
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
kubectl apply -f - <<YAML
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
YAML
kubectl get peerauthentication -A
```

**Expected output:** PeerAuthentication created; sidecars renegotiate to mTLS; a plaintext probe to a meshed pod gets rejected.

**Why it matters:** One YAML resource enforces mutual auth across a whole namespace — this is zero trust as configuration.

---

## Exercise 8: Watch Sidecar Cert Rotation

**Goal:** Prove rotation is automatic and short-lived.

**Commands:**

```bash
kubectl exec deploy/sleep -c istio-proxy -- \
  openssl s_client -connect localhost:15020 2>/dev/null | head -5
# Compare the cert's dates now and again in a few hours:
kubectl exec deploy/sleep -c istio-proxy -- \
  cat /etc/certs/cert-chain.pem | openssl x509 -noout -dates
```

**Expected output:** A cert with a short lifetime (default 24h in Istio), refreshed without any pod restart.

**Why it matters:** Short-lived certs only work because rotation is hands-free — this is the automation behind Q9.

---

## Exercise 9: Lock Down a Namespace with Default-Deny NetworkPolicy

**Goal:** Add the network-layer half of zero trust.

**Commands:**

```bash
kubectl apply -f - <<YAML
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: payments
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
YAML
kubectl get networkpolicy -n payments
```

**Expected output:** All traffic to/from pods in `payments` blocked until explicit allow rules are added.

**Why it matters:** mTLS answers "who are you"; default-deny answers "should you even reach me". Zero trust needs both.

---

## Exercise 10: Break It on Purpose — Clock Skew

**Goal:** See a valid cert rejected because of a wrong clock.

**Commands:**

```bash
# In a container with faketime (or a VM), shift the clock forward:
faketime -f "+40d" openssl verify -CAfile ca.crt server.crt
```

**Expected output:** `certificate has expired` — for a cert that is perfectly valid on the real clock.

**Why it matters:** Clock skew is the sneaky root cause behind "verify failed" tickets; check NTP before blaming PKI.

---

*Completed all 10? You can now debug mTLS failures faster than most on-call rotations.*
