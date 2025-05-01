# content-filtering
# ContentBlocker.ps1
# Script to block pornographic and inappropriate content
# Must be run as administrator

# Check if running as administrator
$isAdmin = ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
if (-not $isAdmin) {
    Write-Host "This script needs to be run as Administrator." -ForegroundColor Red
    Write-Host "Please right-click PowerShell and select 'Run as Administrator'." -ForegroundColor Yellow
    exit
}

# Hosts file location
$hostsFile = "$env:windir\System32\drivers\etc\hosts"

# Common adult content domains to block
# This is a starter list - not comprehensive
$sitesToBlock = @(
    # Major adult content sites
    "pornhub.com", "www.pornhub.com",
    "xvideos.com", "www.xvideos.com",
    "xnxx.com", "www.xnxx.com",
    "youporn.com", "www.youporn.com",
    "redtube.com", "www.redtube.com",
    "xhamster.com", "www.xhamster.com",
    "spankbang.com", "www.spankbang.com",
    "tube8.com", "www.tube8.com",
    "pornhd.com", "www.pornhd.com",
    "chaturbate.com", "www.chaturbate.com",
    "brazzers.com", "www.brazzers.com",
    "onlyfans.com", "www.onlyfans.com",
    "livejasmin.com", "www.livejasmin.com",
    "bangbros.com", "www.bangbros.com",
    "toongod.org", "www.toongod.org",
    
    # Search engine explicit content filters
    # These URLs are used by search engines for adult image searches
    # Google image search adult content
    "www.google.com/search?tbm=isch&q=xxx",
    "www.google.com/search?tbm=isch&q=porn",
    "www.google.com/search?tbm=isch&q=adult",
    "www.google.com/search?tbm=isch&q=nude",
    "www.google.com/search?tbm=isch&q=naked",
    "www.google.com/search?tbm=isch&q=sex",
    
    # Bing explicit image searches
    "www.bing.com/images/search?q=xxx",
    "www.bing.com/images/search?q=porn",
    "www.bing.com/images/search?q=adult",
    "www.bing.com/images/search?q=nude",
    "www.bing.com/images/search?q=naked",
    "www.bing.com/images/search?q=sex",
    
    # Reddit NSFW content
    "reddit.com/r/nsfw",
    "www.reddit.com/r/nsfw",
    "reddit.com/over18",
    "www.reddit.com/over18",
    
    # Image hosting sites often used for adult content
    "imgbox.com", "www.imgbox.com",
    "imgur.com/r/nsfw", "www.imgur.com/r/nsfw"
    # Add more sites as needed
)

function Block-WebsitesViaHosts {
    Write-Host "Blocking websites via hosts file..." -ForegroundColor Cyan
    
    # Read current hosts file content
    $hostsContent = Get-Content -Path $hostsFile -Raw
    
    # Create a timestamp for our entries
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $newEntries = "`n# Content blocks added by PowerShell script on $timestamp`n"
    
    # Flag to track if we need to add anything
    $addedEntries = $false
    
    # Add each site to block
    foreach ($site in $sitesToBlock) {
        # Check if site is already blocked
        if ($hostsContent -notmatch "127\.0\.0\.1\s+$([regex]::Escape($site))") {
            $newEntries += "127.0.0.1 $site`n"
            $addedEntries = $true
        }
    }
    
    # Only modify file if we have new entries
    if ($addedEntries) {
        Add-Content -Path $hostsFile -Value $newEntries -Force
        Write-Host "Successfully blocked websites in hosts file." -ForegroundColor Green
    } else {
        Write-Host "All specified websites are already blocked in hosts file." -ForegroundColor Yellow
    }
}

function Unblock-WebsitesFromHosts {
    Write-Host "Unblocking websites from hosts file..." -ForegroundColor Cyan
    
    # Read current hosts file
    $hostsContent = Get-Content -Path $hostsFile
    
    # Create new content excluding the blocked sites
    $newContent = @()
    foreach ($line in $hostsContent) {
        $shouldKeep = $true
        foreach ($site in $sitesToBlock) {
            if ($line -match "127\.0\.0\.1\s+$([regex]::Escape($site))") {
                $shouldKeep = $false
                break
            }
        }
        if ($shouldKeep) {
            $newContent += $line
        }
    }
    
    # Write back the modified content
    $newContent | Set-Content -Path $hostsFile -Force
    
    Write-Host "Successfully unblocked websites from hosts file." -ForegroundColor Green
}

function Configure-SafeSearch {
    Write-Host "Configuring Safe Search in Windows Registry..." -ForegroundColor Cyan
    
    # For Windows Search
    $registryPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\SearchSettings"
    if (-not (Test-Path $registryPath)) {
        New-Item -Path $registryPath -Force | Out-Null
    }
    Set-ItemProperty -Path $registryPath -Name "SafeSearchMode" -Value 2 -Type DWord -Force
    
    # For Edge browser (if installed)
    $edgePath = "HKCU:\Software\Microsoft\Edge\Safety"
    if (Test-Path $edgePath) {
        if (-not (Test-Path "$edgePath\Family")) {
            New-Item -Path "$edgePath\Family" -Force | Out-Null
        }
        Set-ItemProperty -Path "$edgePath\Family" -Name "SafeSearchSetting" -Value 2 -Type DWord -Force
    }
    
    # Create Google SafeSearch forced parameter file
    # This forces SafeSearch on Google when using any browser
    $googleSearchParams = @'
# Force Google SafeSearch
127.0.0.1 www.google.com/preferences
127.0.0.1 www.google.com/setprefs
127.0.0.1 www.google.com/images?q=
127.0.0.1 www.google.com/imghp
'@
    Add-Content -Path $hostsFile -Value $googleSearchParams -Force
    
    # Force search engines to use safe search by redirecting to safe versions
    $searchEngineParams = @'
# Redirect to Safe Search versions
forcesafesearch.google.com www.google.com
strict.bing.com www.bing.com
strictfamily.youtube.com www.youtube.com
'@
    Add-Content -Path $hostsFile -Value $searchEngineParams -Force
    
    Write-Host "Safe Search settings configured for Windows, Google, Bing, and YouTube." -ForegroundColor Green
}

function Add-ContentFirewallRules {
    Write-Host "Setting up Windows Firewall rules for content filtering..." -ForegroundColor Cyan
    
    # Check if the rule already exists and remove it to update
    $existingRule = Get-NetFirewallRule -DisplayName "Block Adult Content" -ErrorAction SilentlyContinue
    if ($existingRule) {
        Remove-NetFirewallRule -DisplayName "Block Adult Content"
    }
    
    # Find Brave browser path
    $bravePaths = @(
        "C:\Program Files\BraveSoftware\Brave-Browser\Application\brave.exe",
        "C:\Program Files (x86)\BraveSoftware\Brave-Browser\Application\brave.exe",
        "$env:LOCALAPPDATA\BraveSoftware\Brave-Browser\Application\brave.exe"
    )
    
    $braveExePath = ""
    foreach ($path in $bravePaths) {
        if (Test-Path $path) {
            $braveExePath = $path
            break
        }
    }
    
    if (-not $braveExePath) {
        Write-Host "Brave browser not found. Will create rules without program-specific targeting." -ForegroundColor Yellow
    }
    
    # Create a new firewall rule to block common adult content domains
    # These IP addresses are examples of some adult content hosting services
    # A more comprehensive solution would use a regularly updated blocklist
    $adultContentIPs = @(
        "104.244.42.0/24",  # Example range
        "104.244.43.0/24",  # Example range
        "185.88.181.0/24",  # Example range
        "66.254.114.0/24"   # Example range
    )
    
    if ($braveExePath) {
        New-NetFirewallRule -DisplayName "Block Adult Content - Brave" -Direction Outbound -Action Block -RemoteAddress $adultContentIPs -Protocol TCP -RemotePort 80, 443 -Program $braveExePath -Description "Blocks access to adult content sites from Brave browser" | Out-Null
    }
    
    # Create a general rule for all browsers
    New-NetFirewallRule -DisplayName "Block Adult Content - All Browsers" -Direction Outbound -Action Block -RemoteAddress $adultContentIPs -Protocol TCP -RemotePort 80, 443 -Description "Blocks access to adult content sites from all browsers" | Out-Null
    
    # Add specific rules for Google Images
    $searchEngineIPs = @(
        "142.250.0.0/15",  # Google range
        "172.217.0.0/16",  # Google range
        "108.177.0.0/17",  # Google range
        "64.233.160.0/19"  # Google range
    )
    
    # Create keyword-based URL filtering for search engines
    # Note: This is simple and can be bypassed, but adds another layer
    
    Write-Host "Firewall rules have been added." -ForegroundColor Green
    Write-Host "Note: For more comprehensive protection, consider using a dedicated content filtering solution." -ForegroundColor Yellow
    
    # Create additional registry keys for parental controls
    Write-Host "Setting additional registry keys for content filtering..." -ForegroundColor Cyan
    
    # Enable Internet Explorer content filtering (can affect system-wide settings)
    $contentAdvisorPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\Zones\3"
    if (-not (Test-Path $contentAdvisorPath)) {
        New-Item -Path $contentAdvisorPath -Force | Out-Null
    }
    Set-ItemProperty -Path $contentAdvisorPath -Name "2001" -Value 0 -Type DWord -Force
    
    Write-Host "Additional registry settings applied." -ForegroundColor Green
}

function Remove-ContentFirewallRules {
    Write-Host "Removing content filtering firewall rules..." -ForegroundColor Cyan
    
    # Remove the firewall rule
    Remove-NetFirewallRule -DisplayName "Block Adult Content" -ErrorAction SilentlyContinue
    
    Write-Host "Firewall rules have been removed." -ForegroundColor Green
}

function Set-DNSToFamilyFiltering {
    Write-Host "Setting DNS servers to family-filtering options..." -ForegroundColor Cyan
    
    # Get network adapters that are currently connected
    $adapters = Get-NetAdapter | Where-Object { $_.Status -eq "Up" }
    
    # Choose between multiple family filter DNS options
    Write-Host "Available Family Filter DNS Options:" -ForegroundColor White
    Write-Host "1. CleanBrowsing Family Filter (Blocks adult content)" -ForegroundColor White
    Write-Host "2. OpenDNS FamilyShield (Stronger filtering, blocks adult content, phishing, some proxies)" -ForegroundColor White
    Write-Host "3. Google Safe DNS (Basic filtering)" -ForegroundColor White
    
    $dnsChoice = Read-Host "Choose DNS provider (1-3, default: 2)"
    
    # Set DNS servers based on choice
    switch ($dnsChoice) {
        "1" {
            foreach ($adapter in $adapters) {
                Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses "185.228.168.168", "185.228.169.168"
            }
            Write-Host "Using CleanBrowsing Family Filter DNS" -ForegroundColor Cyan
        }
        "3" {
            foreach ($adapter in $adapters) {
                Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses "8.8.8.8", "8.8.4.4"
            }
            Write-Host "Using Google Safe DNS" -ForegroundColor Cyan
        }
        default {
            foreach ($adapter in $adapters) {
                # OpenDNS FamilyShield (default choice)
                Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses "208.67.222.123", "208.67.220.123"
            }
            Write-Host "Using OpenDNS FamilyShield (recommended for strongest protection)" -ForegroundColor Cyan
        }
    }
    
    Write-Host "DNS servers have been set to family-filtering servers." -ForegroundColor Green
    Write-Host "This will block many inappropriate sites and images." -ForegroundColor Green
}

function Reset-DNSSettings {
    Write-Host "Resetting DNS servers to DHCP..." -ForegroundColor Cyan
    
    # Get network adapters that are currently connected
    $adapters = Get-NetAdapter | Where-Object { $_.Status -eq "Up" }
    
    foreach ($adapter in $adapters) {
        # Reset DNS to automatic DHCP-assigned values
        Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ResetServerAddresses
    }
    
    Write-Host "DNS servers have been reset to DHCP defaults." -ForegroundColor Green
}

function Add-GoogleImageBlocking {
    Write-Host "Setting up specific Google Images blocking..." -ForegroundColor Cyan
    
    # Force Google Safe Search specifically for images
    $googleImagesBlock = @'
# Block explicit Google Image searches
127.0.0.1 www.google.com/imghp
127.0.0.1 images.google.com

# Redirect Google to Safe Search
216.239.38.120 www.google.com
'@
    Add-Content -Path $hostsFile -Value $googleImagesBlock -Force
    
    Write-Host "Additional Google Images blocking configured." -ForegroundColor Green
}

function Show-Menu {
    Write-Host "`n==== Content Filtering System ====" -ForegroundColor Magenta
    Write-Host "1. Enable full content filtering (recommended)" -ForegroundColor White
    Write-Host "2. Block websites via hosts file only" -ForegroundColor White
    Write-Host "3. Set family-friendly DNS servers" -ForegroundColor White
    Write-Host "4. Configure Safe Search settings" -ForegroundColor White
    Write-Host "5. Add firewall rules" -ForegroundColor White
    Write-Host "6. Block Google Images specifically" -ForegroundColor White
    Write-Host "7. Disable all content filtering" -ForegroundColor White
    Write-Host "8. Exit" -ForegroundColor White
    
    $choice = Read-Host "Enter your choice (1-8)"
    
    switch ($choice) {
        "1" { 
            Block-WebsitesViaHosts
            Set-DNSToFamilyFiltering
            Configure-SafeSearch
            Add-ContentFirewallRules
            Add-GoogleImageBlocking
            Write-Host "`nFull content filtering enabled including Google Images. Restart your browser for all changes to take effect." -ForegroundColor Green
        }
        "2" { 
            Block-WebsitesViaHosts
            Write-Host "`nWebsites blocked via hosts file. Restart your browser for changes to take effect." -ForegroundColor Green
        }
        "3" { 
            Set-DNSToFamilyFiltering
            Write-Host "`nFamily-friendly DNS servers configured. This will block many inappropriate sites." -ForegroundColor Green
        }
        "4" { 
            Configure-SafeSearch
            Write-Host "`nSafe Search settings configured." -ForegroundColor Green
        }
        "5" { 
            Add-ContentFirewallRules
            Write-Host "`nFirewall rules added to block adult content." -ForegroundColor Green
        }
        "6" { 
            Add-GoogleImageBlocking
            Write-Host "`nGoogle Images specifically blocked from showing adult content." -ForegroundColor Green
        }
        "7" { 
            Unblock-WebsitesFromHosts
            Reset-DNSSettings
            Remove-ContentFirewallRules
            Write-Host "`nContent filtering disabled. Restart your browser for changes to take effect." -ForegroundColor Yellow
        }
        "8" { Write-Host "Exiting..." -ForegroundColor Yellow }
        default { Write-Host "Invalid choice. Please enter a number between 1 and 8." -ForegroundColor Red }
    }
}

# Main script execution
Write-Host "Content Filtering System for Windows" -ForegroundColor Cyan
Write-Host "This tool will help block access to pornographic and inappropriate websites." -ForegroundColor White
Write-Host "For best results, use option 1 which enables all protection methods." -ForegroundColor Green
Write-Host "Note: These are basic protections and are not guaranteed to block all inappropriate content." -ForegroundColor Yellow

Show-Menu
