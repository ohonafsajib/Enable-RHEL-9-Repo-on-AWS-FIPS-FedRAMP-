# RHEL 9 on AWS (FIPS / FedRAMP)

## Standard Operating Procedure – Using Red Hat CDN (Non-RHUI)

---

## 1. Purpose

This document defines the **standard, repeatable procedure** for configuring **RHEL 9 EC2 instances in AWS FIPS-enabled / FedRAMP environments** to use **Red Hat CDN repositories** instead of **AWS RHUI**.

This procedure is required to ensure:

- Reliable `dnf` functionality
- Proper TLS validation
- Red Hat supportability
- FedRAMP and FIPS compliance

---

## 2. Scope

This applies to:

- RHEL 9 EC2 instances in AWS
- FIPS-enabled AWS accounts
- FedRAMP or regulated environments
- Systems that must install software from Red Hat CDN

---

## 3. High-Level Flow

1. Remove AWS RHUI
2. Enable subscription-manager repo control
3. Register the system with Red Hat
4. Trust Red Hat Entitlement CA
5. Enable BaseOS and AppStream repos
6. Validate and update

---

## 4. Step-by-Step Procedure

### 4.1 Remove AWS RHUI (Mandatory)

AWS RHEL AMIs are preconfigured to use **RHUI (Red Hat Update Infrastructure)**. RHUI must be removed before switching to Red Hat CDN.

```bash
sudo dnf remove -y rh-amazon-rhui-client*
sudo rm -f /etc/yum.repos.d/redhat-rhui*.repo
```

Verify:

```bash
ls /etc/yum.repos.d/
```

Expected result: directory is empty or contains no `redhat-rhui` files.

---

### 4.2 Enable subscription-manager Repository Control

Ensure Red Hat is allowed to manage repositories.

```bash
sudo sed -i 's/manage_repos *= *.*/manage_repos = 1/' /etc/rhsm/rhsm.conf
```

Verify:

```bash
grep manage_repos /etc/rhsm/rhsm.conf
```

Expected:

```
manage_repos = 1
```

---

### 4.3 Enable subscription-manager DNF Plugin

```bash
sudo sed -i 's/enabled=0/enabled=1/' /etc/dnf/plugins/subscription-manager.conf
```

Verify:

```bash
cat /etc/dnf/plugins/subscription-manager.conf
```

Expected:

```
enabled=1
```

---

### 4.4 Register the System with Red Hat

```bash
sudo subscription-manager unregister || true
sudo subscription-manager clean

sudo subscription-manager register \
  --org <ORG_ID> \
  --activationkey <ACTIVATION_KEY>
```

**Note:** It is normal to see:

```
Overall Status: Disabled
Content Access Mode: Simple Content Access
```

This is expected behavior and not an error.

---

### 4.5 Trust Red Hat Entitlement Certificate Authority (Critical)

RHEL does **not automatically trust Red Hat Entitlement PKI** for system-wide TLS operations. This must be done explicitly.

Verify RHSM certificates:

```bash
ls /etc/rhsm/ca/
```

Expected files:

- `redhat-entitlement-authority.pem`
- `redhat-uep.pem`

Add them to system trust:

```bash
sudo cp /etc/rhsm/ca/redhat-*.pem /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract
```

Verify TLS:

```bash
openssl s_client -connect cdn.redhat.com:443 -verify_return_error
```

Expected:

```
Verify return code: 0 (ok)
```

---

### 4.6 Enable Red Hat CDN Repositories

```bash
sudo subscription-manager refresh

sudo subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms
```

Verify:

```bash
ls -l /etc/yum.repos.d/redhat.repo
```

---

### 4.7 Final Validation

```bash
dnf clean all
dnf repolist
dnf update
```

Expected repositories:

- `rhel-9-baseos`
- `rhel-9-appstream`

---

## 5. FIPS Validation (Optional)

```bash
fips-mode-setup --check
```

Expected:

```
FIPS mode is enabled.
```

---

## 6. Why AWS RHUI Must Be Disabled (Important Explanation)

### 6.1 What RHUI Is

RHUI (Red Hat Update Infrastructure) is an **AWS-managed mirror** of Red Hat repositories. It is designed for **standard AWS accounts**, not for highly regulated or FIPS-restricted environments.

---

### 6.2 Why RHUI Breaks in FIPS / FedRAMP Environments

RHUI fails or becomes unreliable in FIPS/FedRAMP environments due to:

1. **TLS Certificate Chain Differences**\
   RHUI endpoints use AWS-specific certificates that may not align with hardened FIPS trust stores.

2. **Restricted AWS Endpoints**\
   FedRAMP partitions often block or partially allow RHUI backends, resulting in HTTP 403 or metadata failures.

3. **Lack of Entitlement Awareness**\
   RHUI bypasses Red Hat entitlement PKI, which causes conflicts when subscription-manager is also present.

4. **Mutual Exclusion with Red Hat CDN**\
   Red Hat explicitly does **not support mixing RHUI and CDN** on the same system. Doing so causes:

   - Repositories disappearing
   - `Repositories disabled by configuration`
   - `dnf` metadata download failures

5. **Supportability Concerns**\
   In regulated environments, Red Hat support requires direct CDN usage for troubleshooting and compliance audits.

---

### 6.3 Why Red Hat CDN Is the Correct Choice

Using Red Hat CDN:

- Uses **Red Hat’s official entitlement PKI**
- Fully supports **FIPS cryptographic modules**
- Is **FedRAMP-acceptable**
- Is the **only supported model** for hardened, enterprise-controlled environments

---

## 7. Conclusion

AWS RHUI is suitable for default AWS workloads but **must be removed** for FIPS/FedRAMP RHEL systems. Converting to Red Hat CDN and explicitly trusting Red Hat Entitlement CA ensures reliable package management, full compliance, and long-term supportability.

---

## Appendix A – Why `update-ca-trust` Must Be Run *After* Copying Certificates

This appendix explains a common point of confusion encountered during troubleshooting: why running `update-ca-trust` **before** copying certificates has no effect, and why the order of operations matters.

---

### A.1 Understanding the Certificate Locations

RHEL uses two distinct certificate locations with very different purposes:

- `/etc/rhsm/ca/`\
  Stores **Red Hat–specific entitlement certificates** used internally by `subscription-manager`.

- `/etc/pki/ca-trust/source/anchors/`\
  Stores **system-wide trusted Certificate Authorities** used by:

  - OpenSSL
  - curl
  - dnf
  - all TLS-enabled applications

Certificates located only in `/etc/rhsm/ca/` are **not trusted by the operating system** unless explicitly added to the system trust store.

---

### A.2 What `update-ca-trust` Actually Does

The command:

```bash
update-ca-trust extract
```

**does not discover certificates automatically**.

Instead, it:

1. Reads certificates already present in:
   ```
   /etc/pki/ca-trust/source/anchors/
   ```
2. Rebuilds the compiled trust database in:
   ```
   /etc/pki/ca-trust/extracted/
   ```
3. Makes those certificates available to OpenSSL and all system TLS consumers

If no new certificates exist in `anchors/`, the command completes successfully but performs **no functional change**.

---

### A.3 Why the First Attempt Failed

When the following command was run initially:

```bash
update-ca-trust
```

The Red Hat Entitlement CA was **not yet present** in the system trust anchors. As a result:

- The trust store was rebuilt
- No new CA was added
- TLS validation against `cdn.redhat.com` continued to fail

This behavior is expected and correct.

---

### A.4 Why the Second Attempt Worked

The fix became effective only after the entitlement certificates were copied:

```bash
cp /etc/rhsm/ca/redhat-entitlement-authority.pem /etc/pki/ca-trust/source/anchors/
cp /etc/rhsm/ca/redhat-uep.pem /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```

At this point:

- The Red Hat Entitlement Master CA became a **trusted root CA**
- OpenSSL could validate the full certificate chain
- TLS verification succeeded

This was confirmed by:

```bash
openssl s_client -connect cdn.redhat.com:443 -verify_return_error
```

Result:

```
Verify return code: 0 (ok)
```

---

### A.5 Key Takeaway

> `update-ca-trust` activates trust for certificates that already exist in the trust anchors. It does nothing if the certificates have not yet been added.

Correct order is mandatory:

1. Copy certificate → `anchors/`
2. Run `update-ca-trust`
3. Verify with OpenSSL

---

**Document Owner:** Fortinet MIS\
**Intended Audience:** Cloud Security, Platform Engineering, Compliance Teams

