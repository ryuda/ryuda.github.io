Get-ADComputer -Filter * -Properties IPv4Address |
Where-Object {$_.IPv4Address -eq "192.168.0.50"} |
Select-Object Name, IPv4Address