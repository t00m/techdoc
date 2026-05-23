---
Author: Tomás Vírseda
Category: Procedure
Command: find, grep, java
Date: 2014-05-22
DocType: Tutorial
Filesystem: /usr/sap/BO2
OS: Unix
Product: SAP BusinessObjects BI, SAP JVM, OpenSSL
Protocol: SSH, SSL
Scope: SAP Basis
Tag: certificate, keystore, pkcs12
Topic: Security
---

# Generate Keystore and Certificate for SAP BO BI4.0

## Overview

This tutorial explains how to generate a PKCS12 keystore and a self-signed SSL certificate
for SAP BusinessObjects BI 4.0 / 4.1 using the bundled `PKCS12Tool.jar` utility.
The procedure covers locating the tool, checking the SAP JVM version, and running the
tool with the appropriate options to produce the keystore and certificate files.

> **NOTE:** According to the BI 4.1 installation guide (section 8.13.2, step 7), the
> Certificate Signing Request (CSR) can be self-signed - there is no need to submit it
> to an external Certificate Authority (CA).

---

## Prerequisites

- SSH access to the SAP BusinessObjects host (`sapsrvbo2`).
- The SAP BO installation is located under `/usr/sap/BO2`.
- Sufficient OS-level permissions to read the installation directory and run the SAP JVM.

---

## Step 1 - Locate PKCS12Tool

Search the SAP BO installation tree for the `PKCS12Tool.jar` file:

```bash
find /usr/sap/BO2 -name '*.*' | grep -i PKCS12Tool
```

Expected output:

```
/usr/sap/BO2/sap_bobj/enterprise_xi40/java/lib/PKCS12Tool.jar
```

The `PKCS12Tool.jar` is located in the Java library directory of the BO installation:

```
/usr/sap/BO2/sap_bobj/enterprise_xi40/java/
```

---

## Step 2 - Verify the SAP JVM Version

Before running the tool, confirm which Java runtime is available in the SAP installation:

```bash
/usr/sap/BO2/sap_bobj/enterprise_xi40/aix_rs6000_64/sapjvm/bin/java -version
```

Expected output (example):

```
java version "1.6.0_37"
Java(TM) SE Runtime Environment (build 6.1.044)
SAP Java Server VM (build 6.1.044 21.1-b02, Nov  2 2012 03:57:47 - 61_REL - optU - aix ppc64 - 6 - bas2:182728 (mixed mode))
```

---

## Step 3 - Review PKCS12Tool Options

Run the tool without arguments to display usage information:

```bash
/usr/sap/BO2/sap_bobj/enterprise_xi40/aix_rs6000_64/sapjvm/bin/java \
  -jar /usr/sap/BO2/sap_bobj/enterprise_xi40/java/lib/PKCS12Tool.jar
```

Output:

```bash
Usage: PKCS12Tool <options>
       -keystore <filename (keystore.p12)>
       -alias <key entry alias (default)>
       -storepass <keystore password (default)>
       -dname <certificate subject DN (CN=COM)>
       -validity <number of days (3650)>
       -cert <filename (cert.der)>
        (No certificate is generated when importing a keystore)
       -disablefips
       -importkeystore <filename>
       -?
```

---

## Step 4 - Generate the Keystore and Self-Signed Certificate

Run `PKCS12Tool` with the required parameters. Adjust the `-dname`, `-alias`,
`-storepass`, and `-validity` values to match your environment:

```bash
/usr/sap/BO2/sap_bobj/enterprise_xi40/aix_rs6000_64/sapjvm/bin/java \
  -jar /usr/sap/BO2/sap_bobj/enterprise_xi40/java/lib/PKCS12Tool.jar \
  -keystore keystore.p12 \
  -alias default \
  -storepass <your-password> \
  -dname "CN=<your-hostname>" \
  -validity 3650 \
  -cert cert.der
```

| Option | Description |
|---|---|
| `-keystore` | Output PKCS12 keystore file name (e.g. `keystore.p12`). |
| `-alias` | Key entry alias inside the keystore. |
| `-storepass` | Password protecting the keystore. |
| `-dname` | Certificate subject Distinguished Name (e.g. `CN=sapsrvbo2`). |
| `-validity` | Certificate validity in days (default 3650 = 10 years). |
| `-cert` | Output DER-encoded certificate file (e.g. `cert.der`). |

> **NOTE:** The `-disablefips` flag can be added if FIPS mode is not required in your
> environment.

---

## Step 5 - Self-Sign the Certificate (BI 4.1 Procedure)

As described in the BI 4.1 guide (section 8.13.2, step 7), once you have obtained the
CA certificate and private key, and after generating the CSR, you can sign the certificate
yourself - no external CA submission is needed. Follow the on-screen prompts of the
PKCS12Tool or the SAP BI 4.1 SSL configuration guide to complete the signing step.

---

## Summary

After completing this tutorial you will have:

- A PKCS12 keystore file (`keystore.p12`) containing your private key and self-signed certificate.
- A DER-encoded certificate file (`cert.der`) ready to be imported into the SAP BO
  trusted store or distributed to clients.

Refer to the SAP Help Portal and the BI 4.1 SSL configuration guide for the next steps
on importing the keystore into the SAP BusinessObjects Central Management Server (CMS)
and enabling HTTPS.

---

## References

- SAP SCN Wiki - [Generate keystore and certificate for SAP BO BI4.0](http://wiki.scn.sap.com/wiki/display/BOBJ/Generate+keystore+and+certificate+for+SAP+BO+BI4.0)
- SAP Help Portal - NetWeaver 7.0 EHP1 SSL Configuration
