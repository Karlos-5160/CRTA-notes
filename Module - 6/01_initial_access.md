# Initial Access
    External IP range in scope: 192.168.80.0/24
    Internal IP range in scope: 192.168.98.0/24


    • Scanned the both ip ranges and obviously only the external showed the results 
    • found a http service, ssh service running on the exxternal ip
    • intercepted the request in burp suite 
    • found one parameter did command injection and it was succesfull
    • got hard coded plain text creds in the /etc/passwd file (not encrypted)
    • tried ssh with that creds
    • ssh login succesfull
    