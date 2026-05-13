# potatomesh-altitude-alert
# ✈️ PotatoMesh Altitude Alert

A Python script that monitors a [PotatoMesh](https://github.com/l5yth/potato-mesh) instance for suspiciously high-altitude Meshtastic nodes and cross-references them against live flight data — because sometimes someone brings their radio on a plane.

When a node above your altitude threshold is detected, it sends a Meshtastic message to a destination node with the flight callsign, route, and location via [OpenSky Network](https://opensky-network.org/).

---

## How it works

1. Polls your PotatoMesh instance for node telemetry
2. Flags any node above the altitude threshold (default: 800m)
3. Reverse-geocodes the node's coordinates to a human-readable city
4. Queries OpenSky Network for nearby aircraft at a matching altitude
5. If a match is found, looks up the flight's route (origin → destination)
6. Sends a formatted alert via Meshtastic TCP to your destination node

---

## Requirements

### Python packages

```bash
pip install requests meshtastic airportsdata
```

### External services

- A running [PotatoMesh](https://github.com/l5yth/potato-mesh) instance with API access
- A Meshtastic node accessible over TCP (via Wi-Fi)
- Internet access for [OpenSky Network](https://opensky-network.org/) and [Nominatim](https://nominatim.org/) (no API keys required)

---

## Configuration

Edit the constants at the top of `altitude_alert.py`:

```python
INSTANCE  = "http://YOUR_POTATOMESH_IP:41447"   # Your PotatoMesh instance URL
MESH_HOST = "YOUR_MESHTASTIC_NODE_IP"            # IP of your Meshtastic node
THRESHOLD = 800                                  # Altitude threshold in meters
DEST_NODE = 0000000000                           # Destination node ID (see below)
```

### Finding your DEST_NODE ID

The `DEST_NODE` value is the **decimal node number** of the Meshtastic node you want to receive alerts.

**Method 1 — Meshtastic CLI**

```bash
pip install meshtastic
meshtastic --host YOUR_NODE_IP --nodes
```

Look for the `num` field in the output. That's your decimal node ID.

**Method 2 — Meshtastic Python**

```python
import meshtastic
import meshtastic.tcp_interface

iface = meshtastic.tcp_interface.TCPInterface("YOUR_NODE_IP")
for node_id, node in iface.nodes.items():
    print(node_id, node.get("num"), node.get("user", {}).get("longName"))
iface.close()
```

**Method 3 — PotatoMesh web UI**

Open your PotatoMesh instance in a browser and find the node in the node table. The node ID displayed there is in hexadecimal (e.g. `!bf6e1e62`). Convert it to decimal:

```python
int("bf6e1e62", 16)  # → 3211659362
```

Or use any hex-to-decimal converter online.

---

## Usage

```bash
python altitude_alert.py
```

The script runs in a loop, polling every 60 seconds. Each unique node ID is only alerted once per run (tracked in the `seen` set), so you won't get spammed if the node stays aloft.

---

## Example alert

```
✈️ Airline Meshtastic Node Seen!
WN1234 is on flight WN1234
from MDW (Chicago) to LGA (New York City)
Near South Bend, Indiana
Current altitude: 9843m
```

---

## Notes

- OpenSky Network's free API has rate limits. If you're running this continuously, be mindful of how often altitude spikes occur in your area.
- The flight match uses a ±500m altitude tolerance and finds the closest aircraft within a 1° bounding box around the node's coordinates. Matches aren't guaranteed.
- Nominatim is a free geocoding service. The `User-Agent` header in the request is required by their usage policy — don't remove it.

---

## License

Apache 2.0 — same as the upstream [PotatoMesh](https://github.com/l5yth/potato-mesh) project. See `LICENSE` for details.
