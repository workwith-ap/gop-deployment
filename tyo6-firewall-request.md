# Firewall Request — TYO6 RLE WIFI-TH Sensors → GOP Collector

**Requested by:** [your name]
**Date:** [fill in]
**Site:** TYO6
**Priority:** [confirm — this blocks GOP temp/humidity monitoring go-live for TYO6]

## Rule requested

| Field | Value |
|---|---|
| Direction | Inbound |
| Source | 75 hosts, `10.2.226.171`–`10.2.226.245` (contiguous range — see `tyo6-th-sensor-ips.txt` for the literal list if your firewall requires individual entries) |
| Destination | GOP worker node IP(s) at TYO6 — [fill in once cluster networking/NodePort vs LB decision is finalized] |
| Port / Protocol | UDP / 3184 |
| Justification | RLE WIFI-TH environmental sensors (temperature/humidity, data hall cold-aisle monitoring) — one-way push, no return traffic required |

## Notes for the reviewer

- These are the same 75 physical devices already installed and running at TYO6 (see `TH` and `Mapping File` tabs of `Project_Horus_File_TYO6.xlsx` for the full device/location inventory) — this is not new hardware, only a new/changed destination for existing sensor traffic.
- Source list is fixed and known (not a broad subnet request) — all 75 addresses are itemized in the mapping file, one device per IP.
- Traffic is one-way UDP only; no inbound response path is required from the destination back to the sensors.

## Open item before this can be marked complete

Need to confirm with the TYO6 facility/BMS team what destination IP:port these 75 devices are *currently* configured to send to (if any), since this may be a device-reconfiguration task rather than a pure network-path change.
