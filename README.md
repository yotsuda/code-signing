# yotsuda Code Signing

Public certificate (`.cer`) used to Authenticode-sign release binaries
across all yotsuda OSS projects:

- [PowerShell.MCP](https://github.com/yotsuda/PowerShell.MCP)
- [ripple](https://github.com/yotsuda/ripple)
- (others — see project READMEs for which binaries are signed)

The certificate is **self-signed** (not issued by a public CA). The private
key is held only by the project maintainer; this repository contains the
public key only.

## Certificate details

| Field | Value |
|---|---|
| Subject | `CN=yotsuda, O=Yoshifumi Tsuda, C=JP` |
| Issuer | self (same as Subject) |
| Key | RSA 4096, SHA-256 |
| Validity | 2026-04-18 to 2036-04-18 |
| Thumbprint (SHA-1) | `74E5208228DFB12A067747D536BF497B6E98C73C` |
| Thumbprint (SHA-256) | `ABCE0AFEE35BD19EE1DF8F16E64436439516DDC3FD40229EA7786A8B23BC8013` |

The thumbprints are also published in every signed release's notes — verify
them before trusting the certificate.

---

## Installation

Choose the scenario that matches your environment.

### 1. Personal PC (no WDAC enforcement)

Adds the certificate to your machine's Trusted Publisher store. After this,
signed binaries from yotsuda projects no longer trigger "Unknown publisher"
warnings.

```powershell
# Run as Administrator
Import-Certificate `
    -FilePath yotsuda.cer `
    -CertStoreLocation Cert:\LocalMachine\TrustedPublisher
```

### 2. Active Directory domain (recommended for organizations)

Push the certificate via Group Policy. After `gpupdate`, every domain-joined
machine in the targeted OU automatically trusts the publisher.

1. Open **Group Policy Management Console** (`gpmc.msc`) and edit the GPO that
   targets the OU you want.
2. Navigate to:

   ```
   Computer Configuration
     → Policies
       → Windows Settings
         → Security Settings
           → Public Key Policies
             → Trusted Publishers
   ```

3. Right-click → **Import** → select `yotsuda.cer`.
4. Link the GPO to the target OU. After `gpupdate /force` (or the next
   refresh), every machine has the certificate in its
   `LocalMachine\TrustedPublisher` store.

This handles AppLocker publisher rules, SmartScreen "unknown publisher"
warnings, and signed PowerShell script execution. **WDAC / Device Guard is a
separate layer** — see the next section if your policy enforces WDAC.

### 3. WDAC / Device Guard (Enforce mode)

WDAC ignores per-machine trust stores and only honors signers listed in the
policy XML. Adding the certificate to the policy means future updates of
yotsuda OSS binaries pass WDAC without per-version hash exceptions.

```powershell
# Add a User-mode signer rule to your existing WDAC policy XML, derived
# directly from the .cer
Add-SignerRule `
    -FilePath YourExistingPolicy.xml `
    -CertificatePath yotsuda.cer `
    -User

# Convert the updated policy XML to the binary form WDAC enforces
ConvertFrom-CIPolicy `
    -XmlFilePath YourExistingPolicy.xml `
    -BinaryFilePath SiPolicy.p7b
# Then deploy SiPolicy.p7b via your usual mechanism (GPO / Intune / etc.)
```

> **Note:** if your WDAC policy is configured to reject all third-party
> publishers (only Microsoft signatures + explicit hashes are allowed),
> this won't help — and unfortunately a paid CA cert wouldn't help either
> in that case unless the CA's root is in your policy's trust list.

---

## Verifying a signed binary

```powershell
Get-AuthenticodeSignature path\to\binary.exe |
    Format-List Status, SignerCertificate, TimeStamperCertificate
```

The signer thumbprint should match the value listed under
[Certificate details](#certificate-details). If it doesn't, **do not trust
the binary** and please open an issue.

---

## Why self-signed?

- A CA-issued cert costs several hundred USD/year — unsustainable for a
  one-person OSS effort
- WDAC environments that accept publisher rules treat self-signed and
  CA-issued certs the same way once trusted: a single one-time setup
  replaces per-version hash whitelists
- For SmartScreen reputation and "Verified Publisher" UAC, a CA cert
  (especially EV) is more polished, but isn't required to make the
  binaries usable in enterprise deployments

If a sponsor wants to fund a CA-issued (or EV) cert for these projects,
please open an issue.

---

## License

Public-domain. The certificate file is provided "as is" with no warranty.
