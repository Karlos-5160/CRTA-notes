# Kerberos Delegation
    • It allows an authenticated domain user credentials to be re-used to access resources hosted on a
    different server in a domain.
    • This utility is useful in multi-tier applications or architecture.
    • For Example: A domain user authenticates to a Application Server and the Application Server makes a call to the Database Server. The Application Server can request access to resources of the Database
    Server as the domain user (user is impersonated) and not as Application Server service account.
    • The service account for Application Server must be trusted for delegation to be able to make requests
    as an authenticated domain user.

# Kerberos Delegation process
    1. Domain User requests for TGT to DC by
    verifying credentials 

    2. DC provides the user with TGT

    3. User requests a TGS for a service on
    Application Server to DC

    4. DC gives TGS for accessing service on
    Application Server

    5.User sends TGT and TGS to the
    Application Server
    
    6. Application Server service account use
    the user's TGT to request a TGS for DB
    Server from DC

    7. Application Server service account
    connects to the DB server as Domain
    user

# Types of Kerberos Delegation :
    • Unconstrained Delegation : It allows the Application Server to request access to
    ANY service on any server in the domain.

    • Unconstrained Delegation is by-default enabled on Domain Controllers.

    • Constrained Delegation : It allows the Application Server to request access to
    ONLY specified services on specific servers.

# Domain Trusts
    • Trust represents relationship between two domains or forests which allows the users/services of one
    domain or forest to access resources in the other domain or forest.
    • Types of Trust :
    • Parent-Child trust relationship
    • Forest to Forest Trust relationship
    • Tree-root Trust relationship
    • Trust identifies the entitie in a domain or Forest.

# Authorization in Active Directory
    • Authorization means if a user is specifically permitted or denied to access a resource in the AD
    network.

    • AD validates access to a resource based on the user's security token.

    • This security token is the procedure of checking whether a user is a part of Access Control List (ACL) for the requested object.

    • Security token comprises of :
        •User Rights
        • Group SID
        • Individual SID

    • The primary means through which a security principal is identified when trying to access any securable object is an identifier called security identifier (SID) which is unique for each user or security group.

1. whenever a user logs in he/she gets a token that contains informartion like user rights, group sid and individual sid.

# Access control Lists 
    Discretionary Access Control List (DACL)
        User accounts, groups that are
        allowed or denied access

    The System Access Control List (SACL)
        Defines operations such as read,
        write or delete that should be
        audited for a user or group.