# NTLM Authentication Reflection

https://decoder.cloud/2025/11/24/reflecting-your-authentication-when-windows-ends-up-talking-to-itself/

**CVE-2025-33073**: creating dns record coercing the target to authenticate and performing the authentication relaying seamlessly.

[signed lab](NTLM%20Authentication%20Reflection/signed%20lab.md)

## **What Authentication Reflection Actually Is**

Authentication reflection is deceptively simple:

1. A victim (machine or user) is coerced into authenticating to an attacker-controlled endpoint.
2. The attacker reflects that authentication attempt back to a service the victim host runs itself.
3. The victim ends up authenticating to its own service using its own identity.

If the victim is a domain-joined Windows machine, the identity used is the computer account , effectively granting **NT AUTHORITY\SYSTEM** on that host.

![image.png](NTLM%20Authentication%20Reflection/image.png)

Depending on the coercion vector, this results in:

➔ **Local Privilege Escalation (LPE)**

➔ **Remote Privilege Escalation (RPE)**

➔ **Full domain compromise** when the reflected authentication hits LDAP/LDAPS on a DC under certain conditions

Reflection attacks don’t steal credentials.

***They make Windows authenticate to itself, on your behalf.***

## **Kerberos Reflection: HardeR, Not Impossible**

Kerberos does not have a universal reflection-detection mechanism, but requiring control of the target SPN and valid Kerberos tickets makes such attacks substantially harder to exploit in practice.

Ok, I said, harder but for sure not impossible 😉