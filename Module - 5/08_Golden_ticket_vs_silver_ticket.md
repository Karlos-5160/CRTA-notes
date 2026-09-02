# 🎫 Kerberos Ticket Forgery: Silver Ticket vs. Golden Ticket 

Both **Golden Ticket** and **Silver Ticket** attacks are Kerberos-based attacks in **Active Directory**. The easiest way to remember them is:

> 🟨 **Golden Ticket = forge a TGT → potentially access almost anything in the domain**
> 🥈 **Silver Ticket = forge a service ticket (TGS) → access a particular service**

### First, understand normal Kerberos authentication

In simplified form:

```text
Client
  │
  │  1. Request TGT
  ▼
Domain Controller / KDC
  │
  │  2. TGT
  ▼
Client
  │
  │  3. Request service ticket using TGT
  ▼
KDC
  │
  │  4. TGS / Service Ticket
  ▼
Client
  │
  │  5. Present ticket to service
  ▼
Server / Service
```

There are two important tickets:

* **TGT (Ticket Granting Ticket)** → used to request tickets for services.
* **TGS / Service Ticket** → used to access a specific service such as SMB, HTTP, MSSQL, etc.

---

## 🟨 Golden Ticket Attack

A Golden Ticket is a **forged TGT**.

The attacker needs the secret associated with the domain's **KRBTGT account**. The KRBTGT account is used by the KDC to sign/encrypt Kerberos TGTs.

Conceptually:

```text
Attacker obtains KRBTGT secret
             │
             ▼
      Forge fake TGT
             │
             ▼
     Present fake TGT
             │
             ▼
 Request service tickets
             │
             ▼
Access domain resources
```

### Why is it powerful?

Because the attacker can create a TGT that claims:

```text
User = Administrator
Groups = Domain Admins
...
```

The domain controller may accept the forged TGT as legitimate if it has been constructed correctly and the relevant KRBTGT secret is valid.

So the attacker can potentially impersonate highly privileged users and access many services across the domain.

### Important point

**Golden Ticket → KRBTGT hash/key**

It is primarily a **domain-level compromise**.

---

# 🥈 Silver Ticket Attack

A Silver Ticket is a **forged service ticket (TGS)**.

Instead of compromising the KRBTGT account, the attacker needs the secret/key of the **computer or service account** associated with the target service.

For example:

```text
Attacker
   │
   │ obtains service account/computer account secret
   ▼
Forge TGS for a particular service
   │
   ▼
Present ticket directly to target service
   │
   ▼
Access that service
```

For example, if targeting SMB:

```text
Attacker
   │
   └── Forge SMB service ticket
             │
             ▼
        Target server
             │
             ▼
             SMB
```

The important distinction is that the Silver Ticket is **service-specific**.

---

# Golden vs Silver

| Feature         | Golden Ticket                         | Silver Ticket                                         |
| --------------- | ------------------------------------- | ----------------------------------------------------- |
| Forged ticket   | **TGT**                               | **TGS / Service Ticket**                              |
| Secret needed   | **KRBTGT** secret                     | Target service/computer account secret                |
| Scope           | Broad/domain-wide potential           | Usually specific service/server                       |
| Created for     | Kerberos authentication generally     | Particular service                                    |
| KDC involvement | TGT can subsequently be used with KDC | Forged TGS can often be presented directly to service |
| Impact          | Very high                             | High but more limited                                 |
| Main target     | Domain                                | Specific service                                      |

### Memory trick 🧠

Think of a hotel:

**Golden Ticket**

> You forge the hotel's **master access pass** → it can be used to obtain access to many rooms.

**Silver Ticket**

> You forge a **specific room's/service pass** → it works for that particular room/service.

---

## Why Golden Ticket is generally more dangerous

Suppose an attacker compromises the **KRBTGT secret**:

```text
KRBTGT compromised
       ↓
Forge TGT
       ↓
Impersonate privileged identities
       ↓
Request tickets for many services
       ↓
Potentially move throughout the domain
```

With a Silver Ticket:

```text
Service account secret compromised
       ↓
Forge ticket
       ↓
Target particular service
       ↓
Access that service
```

So:

**Golden = domain-level ticket forgery**

**Silver = service-level ticket forgery**

One more important distinction for your AD/CRTA studies: **Golden Ticket attacks target the Kerberos trust mechanism at the KDC/TGT level, while Silver Ticket attacks target the service-ticket level.**
