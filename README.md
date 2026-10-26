# Enterprise Two-Tier PKI Lab with Multi-Protocol Enrollment and CLM

A hands-on, production-style Microsoft PKI built on **AD CS**, integrated with an **on-premises Active Directory**. It starts with a secure two-tier hierarchy and certificate autoenrollment, then extends the same CA with **NDES/SCEP (Intune)**, **ACME**, and **Sectigo Certificate Lifecycle Management (CLM)**.

---

## Highlights

- Two-tier AD CS hierarchy: **offline standalone Root CA** and **online Enterprise Issuing CA**, both in Azure
- Issuing CA **domain-joined to on-prem AD** (hybrid identity and certificate issuance)
- **Autoenrollment via GPO** for domain-joined computers and users
- **Single CRL / OCSP server** (CRL, AIA and Online Responder over HTTP) checked by every client for every CA-issued certificate
- **Multiple enrollment paths on one CA:** GPO autoenroll, NDES/SCEP with Intune, ACME, Sectigo CLM
- Troubleshooting write-ups for issues found during the build
- Planned: **Entrust nShield HSM** protection for Root and Issuing CA keys

---

## Architecture

![SS](Screenshots/PKI-2tier-Page-2.png)

---

### Components

| Component | Role | Protocol |
|---|---|---|
| Offline Root CA | Trust anchor, signs the Issuing CA only | Offline, manual |
| Issuing CA (AD CS) | Issues all certificates, domain-joined | RPC/DCOM (MS-WCCE) |
| On-prem AD / DC | Templates, autoenrollment policy, GPO | LDAP |
| CRL / OCSP server | Hosts CRL and AIA files and answers live OCSP queries | HTTP |
| NDES + Intune connector | SCEP enrollment for Intune-managed devices | SCEP over HTTPS |
| ACME server | Automated certificate issuance for servers | ACME (RFC 8555) over HTTPS |
| Sectigo CLM connector | Lifecycle management, discovery, policy, revocation | Outbound HTTPS |

---

## Project Modules

| # | Module | What it covers |
|---|---|---|
| 1 | Two-tier PKI build | Offline Root CA, Issuing CA, CDP/AIA, CRL and OCSP |
| 2 | Autoenrollment | Certificate templates, GPO, on-prem client validation |
| 3 | NDES / SCEP with Intune | Prerequisites, infra readiness, NDES and Intune connector setup, testing, operations, troubleshooting |
| 4 | ACME with AD CS | ACME server on IIS, gMSA, template, test client, troubleshooting |
| 5 | Sectigo CLM integration | Prerequisites, infra readiness, Microsoft CA connector, testing, operations, troubleshooting |
| 6 | HSM (planned) | Entrust nShield for CA key protection |

---

## Tools and Technologies

- **Windows Server:** Active Directory Certificate Services, certificate templates, CRL, AIA/CDP, Online Responder (OCSP)
- **Identity and endpoint:** Active Directory, Group Policy, Microsoft Intune
- **Protocols:** RPC/DCOM, LDAP, HTTP/HTTPS, SCEP, ACME
- **Platforms:** Microsoft Azure, Sectigo Certificate Manager
- **Tooling:** PowerShell, `certutil`, `certreq`, PKI View (`pkiview.msc`), Event Viewer, MMC snap-ins, Posh-ACME

---

## Key Design Decisions

- **Offline Root CA:** the root key is only used to sign the Issuing CA and publish CRLs, which limits exposure.
- **One CRL / OCSP server over HTTP:** lets any client, domain-joined or not, build chains and check revocation without LDAP access, using CRL or OCSP.
- **Enterprise Issuing CA joined to on-prem AD:** enables templates and autoenrollment for domain clients while the CA runs in Azure.
- **One CA, many enrollment paths:** each method (GPO, SCEP, ACME, CLM) uses a dedicated template and service identity, so permissions stay scoped.
- **CLM as governance layer:** AD CS issues, CLM adds inventory, renewal automation, approvals and central policy without replacing the private CA.

---

## Validation

- `pkiview.msc` shows Root and Issuing CA, CDP, AIA and OCSP locations as healthy
- `certutil -verify -urlfetch` confirms chain building and CRL/AIA/OCSP retrieval from clients
- `gpupdate /force` followed by a certificate appearing in the machine store confirms autoenrollment
- Issuance, revocation checks and revocation tested end to end for each enrollment path (including revoking from the Sectigo console)
- Event Viewer and CA logs reviewed for each failure case below

---

## Troubleshooting Highlights

| Symptom | Cause | Fix |
|---|---|---|
| `0x80094800` when enrolling an enrollment-agent certificate | The duplicated template was not published on the CA | Publish the original Enrollment Agent (Computer) template on the issuing CA |
| Sectigo web server request rejected | Profile key size did not match the requested key | Allow RSA 3072 on the profile |
| Revoke from Sectigo fails with `(0x7) Failed to initialize RPC connection` | Connector computer account had only Read and Enroll on the CA | Grant **Manage CA** and **Issue and Manage Certificates** to the connector server account |

More detail is in each module's `Troubleshooting` section.

---

## Skills Demonstrated

- Designing and deploying a two-tier Microsoft PKI in a hybrid environment
- CDP/AIA placement, CRL and OCSP strategy and revocation configuration
- Group Policy autoenrollment and certificate template design
- SCEP/NDES with Intune, ACME on AD CS, and CLM integration with a private CA
- CA role separation and least-privilege permissions for service accounts
- Systematic PKI diagnostics with Microsoft tooling

---

## Roadmap

- [ ] Protect Root and Issuing CA keys with **Entrust nShield HSM** (offline module for root, network module for issuing)
- [ ] HSM-backed OCSP signing key
- [ ] Certificate template hardening and key-size / algorithm policy review

---

## References

- [Active Directory Certificate Services overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-cs/active-directory-certificate-services-overview)
- [Sectigo Certificate Manager documentation](https://www.sectigo.com/resource-library)
- [ACME protocol, RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)

---
