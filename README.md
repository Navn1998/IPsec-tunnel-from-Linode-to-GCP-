# IPsec-tunnel-from-Linode-to-GCP-
Below is a **clean, Github-friendly README.md** version of your entire setup.
I’ve rewritten it clearly, organized the flow, added headings, code blocks, explanations, and removed duplicates while keeping all technical details **exactly accurate**.

---

# **IPsec VPN Between Linode (Akamai Cloud Compute) and Google Cloud Platform (GCP) Using StrongSwan**

This guide walks through establishing an **IPsec IKEv2 site-to-site VPN** between **Linode (Akamai)** and **GCP**, using **StrongSwan** on the Linode side.
The Linode StrongSwan VM also acts as a NAT gateway for private instances behind a VLAN.

---

## **📌 Network Topology**

### **Linode / Akamai**

| Component                      | Value              |
| ------------------------------ | ------------------ |
| StrongSwan Public IP           | **172.232.121.82** |
| VLAN CIDR                      | **192.168.1.0/24** |
| VPC CIDR                       | **172.16.1.0/24**  |
| StrongSwan LAN/VLAN Gateway IP | **192.168.1.1**    |

---

### **Google Cloud Platform**

| Component                 | Value             |
| ------------------------- | ----------------- |
| GCP VPN Gateway Public IP | **34.47.217.208** |
| GCP VPC CIDR              | **10.2.0.0/20**   |

---

## **🛠️ Prerequisites**

* Linode VM (Ubuntu recommended)
* StrongSwan installed
* VLAN configured on Linode
* GCP VPN Gateway + Tunnel configured
* IPSec Pre-Shared Key generated on GCP

⚠️ **Important:**
Ensure *Linode VLAN/VPC CIDRs do NOT overlap* with the GCP VPC CIDR.

---

# **1. Install StrongSwan on Linode**

```bash
sudo apt update
sudo apt install -y strongswan strongswan-pki net-tools iptables-persistent
```

---

# **2. Configure IPsec (StrongSwan)**

Edit **/etc/ipsec.conf**

```bash
sudo nano /etc/ipsec.conf
```

Paste:

```conf
conn gcp
    auto=start
    keyexchange=ikev2
    type=tunnel
    authby=secret

    left=%defaultroute
    leftid=172.232.121.82
    leftsubnet=192.168.1.0/24

    right=34.47.217.208
    rightsubnet=10.2.0.0/20

    ike=aes256-sha1-modp1024
    esp=aes256-sha1

    ikelifetime=3600s
    lifetime=3600s

    dpdaction=restart
    dpddelay=30s

    rekey=no
    keyingtries=%forever
```

---

# **3. Create the StrongSwan PSK**

Edit `/etc/ipsec.secrets`:

```bash
sudo nano /etc/ipsec.secrets
```

Add:

```conf
172.232.121.82 34.47.217.208 : PSK "<Tunnel1-Pre-Shared-Key>"
```

---

# **4. Enable NAT on StrongSwan Gateway (Linode)**

This ensures that VLAN instances (192.168.1.0/24) reach GCP (10.2.0.0/20).

### **IPsec Policy NAT Exemption**

```bash
iptables -t nat -I POSTROUTING -s 192.168.1.0/24 -d 10.2.0.0/20 \
    -m policy --pol ipsec --dir out -j ACCEPT
```

### **Route GCP network through StrongSwan**

```bash
ip route add 10.2.0.0/20 dev eth0
```

### **Enable forwarding & NAT for outbound traffic**

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -j ACCEPT
```

Make persistent (if using iptables-persistent):

```bash
sudo netfilter-persistent save
```

---

# **5. Configure Linode Private VM (No Public IP)**

Private VM must route traffic via StrongSwan Gateway.

```bash
ip route replace default via 192.168.1.1 dev eth1
```

Where:

* `192.168.1.1` = StrongSwan VLAN Gateway IP
* `eth1` = VLAN interface

---

# **6. Configure Linode VM With Public IP (Optional)**

View routing table:

```bash
ip r l
```

Example output:

```
default via 172.16.1.1 dev eth0 proto static 
172.16.1.0/24 dev eth0 proto kernel scope link src 172.16.1.4 
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.5 
```

Manually route GCP network through StrongSwan:

```bash
ip route add 10.2.0.0/20 via 192.168.1.1 dev eth1
```

---

# **7. Restart StrongSwan**

```bash
sudo systemctl restart strongswan
sudo systemctl status strongswan
```

Check tunnel status:

```bash
ipsec statusall
```

---

# **8. Expected Result**

✔️ Linode VLAN machines (192.168.1.0/24) can reach GCP VPC (10.2.0.0/20)
✔️ GCP can reach Linode VLAN
✔️ StrongSwan VM performs NAT + routing
✔️ Static routes ensure proper forwarding on both sides

---

# **9. Troubleshooting Tips**

| Issue                             | Cause                  | Fix                                               |
| --------------------------------- | ---------------------- | ------------------------------------------------- |
| Tunnel up but traffic not flowing | Missing NAT exemption  | Ensure iptables rule #1 is applied                |
| GCP unreachable                   | Missing route          | Verify `ip route add 10.2.0.0/20 dev eth0`        |
| Private VM unable to ping GCP     | Wrong default gateway  | Ensure `ip route replace default via 192.168.1.1` |
| Tunnel flapping                   | Incorrect lifetime/dpd | Ensure values match GCP configuration             |

---

# **10. Useful Commands**

```
ipsec status
ipsec statusall
journalctl -u strongswan
ip xfrm policy
ip xfrm state
iptables -t nat -L -n -v
```

---
