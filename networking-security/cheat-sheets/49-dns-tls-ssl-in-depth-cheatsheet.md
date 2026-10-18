# DNS & TLS/SSL Cheat Sheet

*Dense command reference — keep it open during interviews and incidents.*

## dig essentials

```bash
dig example.com +short                 # just the answer
dig example.com +trace                 # walk the full referral chain
dig example.com @8.8.8.8               # query a specific resolver
dig example.com @ns1.provider.com      # ask the authoritative server directly
dig -x 93.184.215.14                  # reverse lookup (PTR)
dig example.com ANY +noall +answer     # all record types (often blocked, try per-type)
```

## Record types

```bash
dig example.com A +short        # IPv4
dig example.com AAAA +short     # IPv6
dig www.example.com CNAME +short
dig example.com MX +short       # mail servers
dig example.com TXT +short      # SPF / DKIM / verification
dig example.com NS +short       # authoritative nameservers
dig example.com SOA +short      # zone authority + serial
dig _sip._tcp.example.com SRV +short
```

## DNS debugging

```bash
cat /etc/resolv.conf            # which resolvers am I actually using?
systemd-resolve --status        # systemd-resolved view
resolvectl query example.com   # test through systemd-resolved
nslookup example.com 8.8.8.8
getent hosts myapp.local       # includes /etc/hosts (NSS order!)
```

## DNS cutover checklist

1. Lower TTL to 300 **48h before** the change
2. Make the change on the authoritative server
3. Verify: `dig name @authoritative-ns +short`
4. Spot-check public resolvers: `@1.1.1.1`, `@8.8.8.8`, `@9.9.9.9`
5. Wait out the old TTL, then raise TTL back up

## Certificate inspection

```bash
# leaf cert: subject, issuer, dates
echo | openssl s_client -connect host:443 -servername host 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# full chain validation (0 = ok, 21 = missing intermediate)
echo | openssl s_client -connect host:443 -servername host 2>/dev/null \
  | grep "Verify return code"

# dump the whole chain
echo | openssl s_client -connect host:443 -showcerts 2>/dev/null \
  | awk '/BEGIN CERT/,/END CERT/'

# SANs on the cert
openssl x509 -in cert.pem -noout -ext subjectAltName
```

## TLS version / cipher checks

```bash
# which protocol does the server negotiate?
echo | openssl s_client -connect host:443 -tls1_3 2>/dev/null | grep Protocol
echo | openssl s_client -connect host:443 -tls1_2 2>/dev/null | grep Protocol

# ciphers offered
nmap --script ssl-enum-ciphers -p 443 host
```

## Quick self-signed cert

```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost"
```

## Expiry monitoring

```bash
# days until expiry
end=$(date -d "$(openssl x509 -enddate -noout -in cert.pem | cut -d= -f2)" +%s)
echo $(( (end - $(date +%s)) / 86400 )) days left
```

## Rules of thumb

- **Never** run an open recursive resolver
- **Never** CNAME the zone apex — use ALIAS/ANAME
- Lower TTL **before** migrations, not during
- Serve the **full chain** (leaf + intermediates)
- Enforce **TLS 1.2 minimum**, prefer 1.3
- Alert on cert expiry at **30 and 7 days**
- Terminate TLS at the **edge** (ALB / ingress / CDN), automate renewal (ACM / cert-manager)
