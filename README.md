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

1. Ethernet/iPhone USB - Highest
2. Home wifi - Second
3. Phone hotspot - Lowest


