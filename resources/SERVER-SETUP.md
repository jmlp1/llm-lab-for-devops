# Server Setup: Linux Server or VM

Used from Unit 3 onward (Skills API + Qdrant). You do not need this for Units 1–2.

---

## Requirements

- A Linux server or VM reachable on your LAN (bare metal, VM on a hypervisor, or a cloud VM)
- Ubuntu 22.04 LTS recommended
- At least 4GB RAM, 2 CPUs, 40GB disk
- Static IP or DHCP reservation (so the address doesn't change between reboots)

---

## Step 1: Prepare the Machine

If using a hypervisor (Proxmox, XCP-ng, Hyper-V, VMware etc.):
1. Create a new VM — Ubuntu 22.04 LTS
2. Allocate: 4GB RAM, 2 vCPUs, 40GB disk
3. Connect to your LAN network
4. Complete the Ubuntu install

If using bare metal, just install Ubuntu 22.04 directly.

---

## Step 2: Install Docker

SSH into the server, then run:
```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
# Log out and back in for group change to take effect
```

Verify:
```bash
docker --version
docker run hello-world
```

---

## Step 3: Set a Static IP (or DHCP Reservation)

- Find the MAC address: `ip link show`
- Set a DHCP reservation on your router for that MAC
- Or configure a static IP in `/etc/netplan/` (Ubuntu)

Note the server's IP address — you'll reference it in Unit 3+ config files as `LAB-SERVER-IP`.

---

## Step 4: Firewall Rules

Only allow access from your laptop's IP:
```bash
sudo ufw allow from YOUR_LAPTOP_IP to any port 22
sudo ufw allow from YOUR_LAPTOP_IP to any port 5000
sudo ufw allow from YOUR_LAPTOP_IP to any port 6333
sudo ufw enable
sudo ufw status
```

---

## Step 5: Verify Connectivity from Laptop

```powershell
# After containers are deployed:
curl http://LAB-SERVER-IP:5000/api/system/info   # Unit 3
curl http://LAB-SERVER-IP:6333/health             # Unit 4
```

---

## Services by Unit

| Unit | Service | Port | Notes |
|------|---------|------|-------|
| Unit 3 | Skills API | 5000 | .NET Minimal API in Docker |
| Unit 4 | Qdrant | 6333 | Vector database |
