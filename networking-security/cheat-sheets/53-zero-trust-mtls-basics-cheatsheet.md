# Zero Trust & mTLS — Cheat Sheet

## Concepts in one line

| Term | Meaning |
|---|---|
| Zero trust | Never trust, always verify — per request, per identity |
| mTLS | TLS where client and server both present certificates |
| CA | Authority whose signature makes a cert trustworthy |
| Chain | Root → intermediate → leaf; every link must verify |
| SAN | SubjectAltName — the hostname(s) a cert is valid for |
| SPIFFE | Standard for workload identity (`spiffe://trust-domain/ns/X/sa/Y`) |
| SPIRE | Runtime that issues SPIFFE identities (SVIDs) |
| SVID | SPIFFE Verifiable Identity Document — usually an X.509 cert |
| STRICT mode | Istio PeerAuthentication mode requiring mTLS for all traffic |

## OpenSSL essentials

```bash
# Read a cert
openssl x509 -in cert.crt -noout -subject -issuer -dates -ext subjectAltName

# Check expiry of a live server
echo | openssl s_client -connect host:443 -showcerts 2>/dev/null \
  | openssl x509 -noout -enddate

# Verify chain against your CA
openssl verify -CAfile ca.crt server.crt

# Self-signed CA + sign a cert
openssl req -x509 -newkey rsa:2048 -keyout ca.key -out ca.crt -days 365 -nodes -subj "/CN=my-ca"
openssl x509 -req -in app.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out app.crt -days 30

# mTLS client call
curl --cert client.crt --key client.key https://service.internal:8443/

# mTLS test server (requires client cert)
openssl s_server -accept 8443 -cert server.crt -key server.key -CAfile ca.crt -Verify 1 -www
```

## Handshake order (mTLS)

1. ClientHello → ServerHello + server Certificate
2. **CertificateRequest** (the mTLS extra — server demands client cert)
3. Client sends Certificate + CertificateVerify (signed transcript)
4. Both verify chains → derive session keys → app data flows

## Debug checklist: "certificate verify failed"

1. Expiry: `-enddate` on the cert
2. Chain: is the intermediate included server-side?
3. Trust store: does the client have your CA root?
4. SAN match: does the cert cover the hostname/IP being called?
5. Clock skew: is the client's clock right? (check NTP)

## Istio mTLS

```bash
# Namespace-wide STRICT mTLS
kubectl apply -f - <<YAML
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: {name: default, namespace: payments}
spec:
  mtls: {mode: STRICT}
YAML

# Check effective mTLS status
istioctl x describe pod <pod>
```

## Zero-trust checklist

- [ ] mTLS (or equivalent) on all service-to-service calls
- [ ] Default-deny NetworkPolicies per namespace
- [ ] Short-lived certs with automated rotation (cert-manager / mesh CA)
- [ ] No shared SSH keys — identity-aware access (Teleport/Boundary) + SSO
- [ ] Cert-expiry monitoring with alerts, not spreadsheets
- [ ] SPIFFE/SPIRE (or mesh) identities instead of IP-based trust
