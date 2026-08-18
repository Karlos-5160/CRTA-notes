# Active Directory:
    • As the name suggests, it is a directory (or database) which :
        • Manages the resources of organization like (users, computers, shares etc)
        • Provides Access rules that govern the relationships between these resources.
        • Stores information about objects on the network and makes it available to users and admins.
        • Provides centralized management of all the organization virtual assets.

# Active Directory Forests/Domain :
    • Forest is a single instance of Active Directory.
        • It is basically a collection of Domain Controllers that trust one another.

    • Domains can be thought as containers within the
    scope of a Forest.

    • Organizational Units (OU's) are logical grouping of
    users, computers and other resources

    Groups
        • Collection of users or other groups
        • Privileged, non-privileged
    
    • Active Directory Objects
        • The physical entities that make up an organized network

        • Domain Users
            • User account that are allowed to authenticate to machines/servers in the domain
            
        • Domain Groups (Global Groups):
            • It can be used to assign permissions to access resources in any domain.
        
        • Domaih Computers
            • Machines that are connected to a domain and hence become a member of a domain
        
        • Domain Controller :
            Server located centrally that responds to security authentication requests and manages various resources like computers, users, groups etc.
        
        • Group Policy Objects (GPOs) :
            Collection of policies that are applied to a set of user, domain, domain object etc.
            
        • Ticket Granting Ticket (TGT)
            Ticket used specifically for authentication
        
        • Ticket Granting Service (TGS) :
            Ticket used specifically for authorization

# Kerberos Authentication (port 88)
    • In the Active Directory environment, all the queries and authentication process is don
    through tickets. Hence, no passwords are every travel to network.
    • A ticket is a form of authentication and authorization token and can be categorized as
    follows :
        • Ticket Granting Ticket (TGT) for Authentication
        • Ticket Granting Service (TGS) for Authorization
    • The tickets (TGT and TGS) are stored in memory and can be extracted for abusing
    purposes as these tickets represent user credentials.
    • The TGS can be used for accessing a specific service of a server in the domain.