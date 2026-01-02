# Split Tunneling (Application-Based Filtering)

WireGuard for Windows now supports split tunneling functionality that allows you to control which applications route their traffic through the VPN tunnel based on their executable paths.

## Overview

Split tunneling gives you fine-grained control over which applications use the WireGuard tunnel and which ones bypass it. This can be useful for:

- Routing only specific applications through the VPN while allowing others to use the regular internet connection
- Excluding certain applications from using the VPN (e.g., local network applications, streaming services)
- Optimizing performance by only routing necessary traffic through the tunnel

## Configuration

Two new configuration options are available in the `[Interface]` section:

### IncludedApplications

Specifies a comma-separated list of application executable paths that **should** route their traffic through the WireGuard tunnel. When this option is set, only the listed applications will use the VPN, and all other applications will bypass it.

**Syntax:**
```
IncludedApplications = <path1>, <path2>, ...
```

**Example:**
```ini
[Interface]
PrivateKey = ...
Address = 10.0.0.2/24
IncludedApplications = C:\Program Files\Firefox\firefox.exe, C:\Program Files\Chromium\chrome.exe
```

### ExcludedApplications

Specifies a comma-separated list of application executable paths that **should not** route their traffic through the WireGuard tunnel. When this option is set, the listed applications will bypass the VPN while all other applications use it.

**Syntax:**
```
ExcludedApplications = <path1>, <path2>, ...
```

**Example:**
```ini
[Interface]
PrivateKey = ...
Address = 10.0.0.2/24
ExcludedApplications = C:\Windows\System32\svchost.exe, C:\Program Files\Spotify\Spotify.exe
```

## Usage Guidelines

### Important Notes

1. **Full Path Required**: Always use the complete, absolute path to the executable file (e.g., `C:\Program Files\App\app.exe`).

2. **Mutually Exclusive**: It is recommended to use either `IncludedApplications` OR `ExcludedApplications`, but not both in the same configuration. If both are specified:
   - `IncludedApplications` permits the listed apps (higher priority)
   - `ExcludedApplications` blocks the listed apps (higher priority)
   - The behavior depends on the firewall rule weight ordering

3. **Case Sensitivity**: Paths on Windows are case-insensitive, but it's best practice to match the actual case of the file system.

4. **Requires Full Tunnel**: For application filtering to work effectively, your WireGuard configuration should be set up as a full tunnel (with `AllowedIPs = 0.0.0.0/0` for IPv4 or `::/0` for IPv6).

### Example Configuration

**Only allow web browsers through the VPN:**
```ini
[Interface]
PrivateKey = yAnz5TF+lXXJte14tji3zlMNq+hd2rYUIgJBgB3fBmk=
Address = 10.0.0.2/24
DNS = 10.0.0.1
IncludedApplications = C:\Program Files\Mozilla Firefox\firefox.exe, C:\Program Files\Google\Chrome\Application\chrome.exe

[Peer]
PublicKey = xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
```

**Exclude specific applications from the VPN:**
```ini
[Interface]
PrivateKey = yAnz5TF+lXXJte14tji3zlMNq+hd2rYUIgJBgB3fBmk=
Address = 10.0.0.2/24
DNS = 10.0.0.1
ExcludedApplications = C:\Program Files\Spotify\Spotify.exe, C:\Program Files\Steam\Steam.exe

[Peer]
PublicKey = xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
```

## How It Works

The application filtering feature uses Windows Filtering Platform (WFP) to create firewall rules based on application identifiers (AppIDs). When the WireGuard tunnel starts:

1. The configuration is parsed to extract the list of included or excluded applications
2. For each application path, an AppID is generated using the Windows API
3. Firewall rules are created to either permit or block traffic for those specific applications
4. These rules are applied at a higher priority than the default tunnel routing rules

The rules apply to both IPv4 and IPv6 traffic, and cover both inbound and outbound connections.

## Troubleshooting

### Applications Still Bypassing/Using the VPN

- Verify the exact path to the executable (use Task Manager to find the running process location)
- Ensure the configuration has been saved and the tunnel has been restarted
- Check that the application is launching the correct executable (some apps use launcher processes)
- Review the WireGuard logs for any errors related to firewall rule creation

### Application Path Not Found

- Make sure to use the full, absolute path to the executable
- Verify that the executable file exists at the specified location
- Check for typos in the path
- Ensure proper escaping of special characters if needed

### Performance Issues

- Limit the number of applications in the filter lists to what you actually need
- Consider using `ExcludedApplications` instead of `IncludedApplications` if most applications should use the VPN
- Monitor system resources to ensure the filtering is not causing bottlenecks

## Security Considerations

- Application filtering relies on the executable path, which can potentially be spoofed by advanced malware
- For high-security environments, additional endpoint protection measures should be used
- The filtering is enforced at the application layer and does not prevent all forms of traffic leakage
- Administrative privileges are required to modify firewall rules

## Limitations

- Only applies to applications launched after the tunnel is started
- Does not support wildcards or partial paths
- Cannot filter traffic from Windows services that share executable paths with user applications
- Child processes may or may not inherit the filtering depending on how they're launched
