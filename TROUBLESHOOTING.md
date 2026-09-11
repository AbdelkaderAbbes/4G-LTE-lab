# Troubleshooting Log

Real issues hit while bringing the lab up, in the order they occurred, with root cause and fix. Kept here because the debugging is usually more instructive than the happy path.

---

### 1. `srsenb` — "Error: allocating decimation buffer" / "Error initializing radio"

**Symptom:**
```
Warning: Failed to create thread with real-time priority. Creating it with normal priority: Operation not permitted
...
Error: allocating decimation buffer
Error initializing radio.
```

**Root cause:** the ZMQ RF driver locks its decimation buffer into physical memory for real-time, zero-page-fault operation — the same privilege class as the real-time thread scheduling warning printed immediately above it. Without the right capabilities, that allocation fails hard.

**Fix:** grant the two specific capabilities instead of running everything as root:
```bash
sudo setcap 'cap_sys_nice,cap_ipc_lock+ep' $(which srsenb)
sudo setcap 'cap_sys_nice,cap_ipc_lock+ep' $(which srsue)
```
If that's not enough, check and raise the locked-memory ulimit:
```bash
ulimit -l
# if low, add to /etc/security/limits.conf:
#   <user> soft memlock unlimited
#   <user> hard memlock unlimited
```

---

### 2. `srsLog error - Unable to create log file: Permission denied`

**Symptom:** switching between running `srsenb`/`srsue` as a normal user and with `sudo` leaves a root-owned log file in `/tmp` that the other user can't overwrite.

**Fix:**
```bash
sudo rm -f /tmp/enb.log /tmp/ue.log
```

---

### 3. MME log: `eNB-S1[...] connection refused!!!`

**Symptom:** eNodeB connects to the MME over SCTP, gets accepted, then is dropped again in under a second.

**Investigation:** this pattern (accepted → dropped near-instantly, no S1 Setup Response logged) is the classic signature of a PLMN/TAC mismatch between the eNodeB's config and the MME's Tracking Area list. Turned on debug logging on the MME to confirm:
```yaml
# mme.yaml
logger:
    file: /var/log/open5gs/mme.log
    level: debug
```

**Resolution:** in this case it turned out to be a one-off — a stale SCTP association from a previous `srsenb` run colliding with the new one, not an actual PLMN/TAC mismatch. Confirmed by checking that MCC/MNC in `enb.conf` and TAC in `rr.conf` matched the MME's `tai:` block, and that the *next* connection attempt stayed up indefinitely. Worth checking both in order if this comes up again.

---

### 4. UE attaches and gets an IP, but `ping` from the UE gets 100% packet loss with no error

**Symptom:**
```
ping -I tun_srsue 8.8.8.8
100% packet loss
```
UE shows `Network attach successful. IP: 10.45.0.3` — the LTE side is fully working. The problem is host routing, not the LTE stack.

**Diagnosis path** (each step narrows down where the packet is actually getting dropped):

1. **Confirm the UPF side is receiving the packets at all:**
   ```bash
   sudo tcpdump -i ogstun -n
   ```
   Packets showed up here correctly — confirms the GTP tunnel and UPF are working. The problem is purely in host-level IP forwarding/NAT.

2. **Enable IP forwarding:**
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

3. **Add NAT/masquerade** for the UE subnet going out the real NIC:
   ```bash
   sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
   ```

4. **Rule out UFW** (its default FORWARD policy is DROP even with forwarding enabled and NAT rules in place):
   ```bash
   sudo ufw status verbose
   sudo ufw disable   # or explicitly set DEFAULT_FORWARD_POLICY=ACCEPT and reload
   ```
   In this build, UFW was already inactive — ruled out.

5. **Check the FORWARD chain directly**, since NAT rules only apply to traffic already permitted through it:
   ```bash
   sudo iptables -L FORWARD -n -v --line-numbers
   ```
   Policy was already `ACCEPT`; explicit rules were added anyway for clarity:
   ```bash
   sudo iptables -I FORWARD -i ogstun -o <real-nic> -j ACCEPT
   sudo iptables -I FORWARD -i <real-nic> -o ogstun -m state --state RELATED,ESTABLISHED -j ACCEPT
   ```
   (`<real-nic>` found via `ip route get 8.8.8.8`.)

6. **Reverse-path filtering (rp_filter)** was the next suspect for virtual point-to-point interfaces like `ogstun`/`tun_srsue`, since strict rp_filter commonly blocks return traffic on interfaces that don't have a "normal" route registered:
   ```bash
   sysctl net.ipv4.conf.all.rp_filter
   sysctl net.ipv4.conf.ogstun.rp_filter
   sudo sysctl -w net.ipv4.conf.all.rp_filter=0
   sudo sysctl -w net.ipv4.conf.ogstun.rp_filter=0
   ```

**Status:** narrowing this down packet-hop by packet-hop (tcpdump on `ogstun` → FORWARD counters → rp_filter) rather than guessing was what actually made progress here — the LTE stack itself was correct the entire time; the issue was entirely standard Linux routing/NAT.

---

### 5. `srsenb` — `unrecognised option 'pcap.nas_filename'`

**Symptom:** enabling native pcap capture in `enb.conf` with both `nas_enable`/`nas_filename` set fails to even start.

**Root cause:** the eNodeB has no NAS layer of its own — NAS messages between UE and MME pass through it opaque, wrapped inside S1AP containers. `nas_filename` is a **UE-only** option; `enb.conf`'s `[pcap]` section only supports a single MAC-layer capture.

**Fix — `enb.conf`:**
```ini
[pcap]
enable = true
filename = /tmp/enb_mac.pcap
```
**`ue.conf`** is the one that supports NAS capture:
```ini
[pcap]
enable = true
mac_filename = /tmp/ue_mac.pcap
nas_filename = /tmp/ue_nas.pcap
```

---

### 6. `srsue` — "Failed to setup/configure GW interface" (after a successful attach)

**Symptom:** the LTE attach itself fully succeeds — IMSI authenticated, bearer established, IP allocated — but `srsue` then fails to bring up the local `tun_srsue` interface, so the host can't actually route to that IP.

**Root causes, in order of likelihood:**

1. **A stale `tun_srsue` device left over from a previous run** that wasn't cleanly torn down:
   ```bash
   ip addr show tun_srsue
   sudo ip link delete tun_srsue   # if it exists
   ```
2. **Missing `CAP_NET_ADMIN`.** Creating a TUN device needs this capability. Since `srsue` was only granted `cap_sys_nice,cap_ipc_lock` earlier (for the real-time RF buffer fix), running it un-elevated without `sudo` leaves it unable to create network interfaces:
   ```bash
   sudo setcap 'cap_sys_nice,cap_ipc_lock,cap_net_admin+ep' $(which srsue)
   ```

**Lesson:** the same "attach succeeds at the LTE layer, fails at the Linux layer" pattern as the NAT/routing issue earlier — worth checking `ip addr`/capabilities first before suspecting the LTE stack itself.

---

### 7. iptables rules silently gone after a restart

**Symptom:** routing/NAT that was confirmed working earlier stops working again, with no error — `iptables -t nat -L POSTROUTING` comes back completely empty.

**Root cause:** iptables rules live in kernel memory and don't persist across reboots or certain service restarts by default. Nothing "broke" — the rules were simply never saved.

**Fix:** re-add the rules, then make them persistent:
```bash
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo iptables -I FORWARD -i ogstun -o ens33 -j ACCEPT
sudo iptables -I FORWARD -i ens33 -o ogstun -m state --state RELATED,ESTABLISHED -j ACCEPT

sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

### 8. Running the UE in a network namespace: routing and DNS both need to be set up separately

**Symptom 1 — `ping: connect: Network is unreachable`** when pinging out from inside a namespace (e.g. `ip netns exec ue1 ping 8.8.8.8`), even though pinging the local gateway works fine.

**Root cause:** `srsue` configured with `netns = ue1` in `ue.conf`'s `[gw]` section moves `tun_srsue` into that namespace — and namespaces have their own independent routing table. It only had a route to the local `10.45.0.0/24` subnet, no default route.

**Fix:**
```bash
sudo ip netns exec ue1 ip route add default dev tun_srsue
```

**Symptom 2 — `curl: Could not resolve host`** even after routing is fixed.

**Root cause:** `/etc/resolv.conf` on Ubuntu normally points at `127.0.0.53`, a local stub DNS resolver run by `systemd-resolved` — bound to the **host's** loopback. Loopback is per-namespace, so `127.0.0.53` inside a namespace refers to nothing; there's no resolver listening there.

**Fix:** give the namespace its own resolver config — `ip netns exec` automatically uses `/etc/netns/<name>/resolv.conf` if present:
```bash
sudo mkdir -p /etc/netns/ue1
echo "nameserver 8.8.8.8" | sudo tee /etc/netns/ue1/resolv.conf
```

**Lesson:** a namespace only isolates what you explicitly give it. Interfaces, routes, and DNS resolution all have to be independently provisioned inside it — nothing is inherited from the host just because the namespace exists.

---

## General lesson

Almost none of these were LTE protocol issues. They were Linux fundamentals — capabilities, file permissions, iptables chains, reverse-path filtering — showing up underneath an unfamiliar stack. Verifying at each layer (RF driver → S1AP → NAS → GTP-U → IP forwarding) rather than assuming a fix worked was what made each of these traceable.
