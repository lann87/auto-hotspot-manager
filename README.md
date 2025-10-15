# Automated Hotspot Connection Manager

## Overview

I've discovered a pain point of ubuntu not able to detect iPhone hotspot connection, as it considered a hidden network for Ubuntu whenever I'm away from my desk and back, the connection drops and I have to manually reconnect.
This automated script intelligently prioritizes network connections and automatically connects to iPhone hotspot when other networks are unavailable.

### Problem Solved

- Automatically connect to iPhone's hidden wifi hotspot when away from home
- Prioritize connections: Ethernet (including iPhone USB) > Home wifi > iPhone wifi hotspot
- Reduce manual network switching
- Auto start on login

### Connection Priority

1. **Ethernet/iPhone USB**
2. **Home wifi**
3. **Phone hotspot**

---

## How It Works

### Connection Logic Flow
1. **Check for Ethernet**: If any ethernet connection active (physical or iPhone USB), do nothing
2. **Check for Home WiFi**: If connected to any home WiFi network, do nothing
3. **Check for iPhone Hotspot**: If connected to iPhone WiFi hotspot, monitor only
4. **Attempt Connection**: If no connection, try to connect to iPhone WiFi hotspot
5. **Retry Logic**: 
   - Connected/Stable: Check every 30 seconds
   - Disconnected: Check every 10 seconds
   - Hotspot not found: Wait 120 seconds before retry

### Key Bash Concepts Used
- **Shebang** (`#!/bin/bash`): Tells system to use bash interpreter
- **Variables**: Store reusable values (`HOTSPOT_NAME="AP-21"`)
- **Variable Expansion**: Use `$VARIABLE_NAME` to access stored values
- **Arrays**: Store multiple values (`HOME_WIFI=("wifi1" "wifi2")`)
- **Infinite Loop**: `while true; do ... done` runs forever
- **Conditionals**: `if [ condition ]; then ... else ... fi` for decision making
- **Command Substitution**: `$(command)` captures command output
- **Pipes and Grep**: `|` chains commands, `grep` filters text
- **Sleep**: Pause script execution for specified seconds

---

## Implementation Steps

### 1. Setting network priorities

```bash
# Home wifi - highest priority (100)
nmcli connection modify "dlink-516C-5GHz" connection.autoconnect-priority 100
nmcli connection modify "Network Error" connection.autoconnect-priority 100

# iPhone hotspot - lower priority (50)
nmcli connection modify "AP-21" connection.autoconnect-priority 50
```

### 2. Finding wifi interface name

```bash
# Identify wifi interface
nmcli device status | grep wifi

# Insert the interface name to ifname within connection profile
```

### 3. Create iPhone Hotspot Connection Profile

```bash
# Create hidden network connection profile
nmcli connection add \
  type wifi \
  con-name "Sample_Name" \
  ifname "wlp82s0" \
  ssid "Sample_Name" \
  wifi-sec.key-mgmt wpa-psk \
  wifi-sec.psk "YOUR_PASSWORD_HERE" \
  wifi.hidden yes
```

**Replace:**
- `wlp82s0` - Your WiFi interface name
- `Sample_Name` - Your iPhone hotspot SSID
- `YOUR_PASSWORD_HERE` - Your hotspot password

### 4. Create the Bash Script

Create file: `~/connect-hotspot.sh`

```bash
#!/bin/bash
# Variables
HOTSPOT_NAME="Sample_Name"
HOME_WIFI=("Sample_Name")
CHECK_INTERVAL=10
CONNECTED_INTERVAL=30
NOT_FOUND_INTERVAL=120

echo "Starting smart wifi connection manager..."
echo "Priority: Any Ethernet (including iPhone USB) > Home wifi > iPhone wifi Hotspot"

while true; do
    echo "Checking connection..."
    
    # Check if ANY ethernet is connected (physical or iPhone USB)
    if nmcli device status | grep "ethernet" | grep -q "connected"; then
        ETH_NAME=$(nmcli device status | grep "ethernet" | grep "connected" | awk '{print $1}')
        echo "Connected via Ethernet: $ETH_NAME - no action needed"
        sleep $CONNECTED_INTERVAL
        continue
    fi
    
    # Check if connected to any home wifi
    HOME_CONNECTED=false
    for wifi in "${HOME_WIFI[@]}"; do
        if nmcli connection show --active | grep -q "$wifi"; then
            echo "Connected to home wifi: $wifi"
            HOME_CONNECTED=true
            break
        fi
    done
    
    # If on home wifi, just wait
    if [ "$HOME_CONNECTED" = true ]; then
        sleep $CONNECTED_INTERVAL
        continue
    fi
    
    # Not on ethernet or home wifi, check iPhone wifi hotspot
    if nmcli connection show --active | grep -q "$HOTSPOT_NAME"; then
        echo "Connected to iPhone WiFi hotspot: $HOTSPOT_NAME"
        sleep $CONNECTED_INTERVAL
    else
        echo "Not on ethernet, home wifi, or hotspot. Attempting iPhone wifi hotspot..."
        if nmcli connection up "$HOTSPOT_NAME" 2>&1 | grep -q "could not be found"; then
            echo "Hotspot not available, will retry in $NOT_FOUND_INTERVAL seconds"
            sleep $NOT_FOUND_INTERVAL
        else
            sleep $CHECK_INTERVAL
        fi
    fi
done
```

**Make executable:**
```bash
chmod +x ~/connect-hotspot.sh
```

### 5. Create Systemd User Service

Create directory:
```bash
mkdir -p ~/.config/systemd/user
```

Create file: `~/.config/systemd/user/wifi-manager.service`

```ini
[Unit]
Description=Smart wifi connection manager
After=network.target

[Service]
Type=simple
ExecStart=/home/alann/connect-hotspot.sh
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

**Note:** Replace `/home/alann/` with your actual home directory path

### 6. Enable and Start Service

```bash
# Reload systemd configuration
systemctl --user daemon-reload

# Enable auto-start on login
systemctl --user enable wifi-manager.service

# Start service now
systemctl --user start wifi-manager.service
```

---

## Configuration Variables

Edit `~/connect-hotspot.sh` to customize:

| Variable | Default | Description |
|----------|---------|-------------|
| `HOTSPOT_NAME` | "Sample_Name" | iPhone wifi hotspot connection name |
| `HOME_WIFI` | ("Sample_Name" "Sample_Name") | home wifi network names |
| `CHECK_INTERVAL` | 10 | Seconds between checks when disconnected |
| `CONNECTED_INTERVAL` | 30 | Seconds between checks when connected |
| `NOT_FOUND_INTERVAL` | 120 | Seconds to wait when hotspot not found |

---

## Service Management Commands

### Check Status
```bash
systemctl --user status wifi-manager.service
```

### View Live Logs
```bash
journalctl --user -u wifi-manager.service -f
```

### Restart Service
```bash
systemctl --user restart wifi-manager.service
```

### Stop Service
```bash
systemctl --user stop wifi-manager.service
```

### Disable Auto-Start
```bash
systemctl --user disable wifi-manager.service
```

### Re-enable Auto-Start
```bash
systemctl --user enable wifi-manager.service
```

---

## Troubleshooting

### Script Not Running on Login
```bash
# Check if service is enabled
systemctl --user is-enabled wifi-manager.service

# Should output: enabled
# If not, run:
systemctl --user enable wifi-manager.service
```

### Check for Errors
```bash
# View recent logs
journalctl --user -u wifi-manager.service -n 50

# View logs with timestamps
journalctl --user -u wifi-manager.service --since "10 minutes ago"
```

### Connection Not Working
```bash
# Verify connection profile exists
nmcli connection show | grep "Sample_Name"

# Test manual connection
nmcli connection up "Sample_Name"

# Check WiFi interface name
nmcli device status | grep wifi
```

### Verify Script Path
```bash
# Check which script systemd is using
systemctl --user cat wifi-manager.service | grep ExecStart
```

### Script Not Detecting iPhone USB
```bash
# Check active connections
nmcli connection show --active

# Check device status
nmcli device status

# Should see ethernet device when iPhone USB is connected
```

---

## Future Enhancements (Optional)

- Add desktop notifications for connection changes
- Implement exponential backoff for retry attempts
- Add configuration file instead of editing script
- Log connection events to file for analytics
- Add support for multiple iPhone devices
- Implement connection health checks (ping test)

---

## Author Notes

This solution was built incrementally to learn bash scripting fundamentals while solving a real-world automation problem. The script demonstrates practical DevOps/SRE skills including:
- Service automation
- Network management
- Systemd integration
- Error handling and retry logic
- Logging and monitoring












