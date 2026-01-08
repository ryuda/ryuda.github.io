foreach ($pc in $pcs) {
    try {
        $nics = Get-WmiObject Win32_NetworkAdapterConfiguration -ComputerName $pc -ErrorAction Stop |
                Where-Object {$_.IPAddress -ne $null}

        foreach ($nic in $nics) {
            if ($nic.IPAddress -contains "192.168.0.50") {
                Write-Host "IP Match → $pc"
            }
        }
    } catch {}
}