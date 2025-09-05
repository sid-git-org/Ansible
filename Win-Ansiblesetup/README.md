# Enable WinRM
winrm quickconfig -q

# Allow remote access from any IP (basic setup)
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
winrm set winrm/config/service/auth '@{Basic="true"}'

# Create firewall rule
netsh advfirewall firewall add rule name="WinRM" dir=in action=allow protocol=TCP localport=5985
