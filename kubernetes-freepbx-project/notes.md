## FreePBX Home Lab: RHEL 9 & K3s Implementation Guide
Date: March 17, 2026

Status: Operational

Infrastructure: RHEL 9, K3s (Kubernetes), Longhorn (Storage), Proxmox (Hypervisor)

### 1. Prerequisites & Environment
OS: Red Hat Enterprise Linux 9.x

Kernel: RP_Filter and Firewalld active.

Storage Engine: Longhorn v1.6.0+

PBX Image: tiredofit/freepbx:latest

### 2. Critical RHEL 9 System Hardening
RHEL's default security prevents Kubernetes pod-to-pod communication. These steps were required to allow the Longhorn storage and SIP signaling to function.

A. Networking & Firewall

Bash
# Trust the Flannel/CNI interfaces
sudo firewall-cmd --permanent --zone=trusted --add-interface=flannel.1
sudo firewall-cmd --permanent --zone=trusted --add-interface=cni0

# Open standard K3s and VOIP ports
sudo firewall-cmd --permanent --add-port=6443/tcp     # K8s API
sudo firewall-cmd --permanent --add-port=8472/udp     # Flannel VXLAN
sudo firewall-cmd --permanent --add-port=9500/tcp     # Longhorn UI/Backend
sudo firewall-cmd --permanent --add-port=5060/udp     # SIP Signaling
sudo firewall-cmd --permanent --add-port=10000-20000/udp # RTP (Voice)

sudo firewall-cmd --reload
B. Reverse Path Filtering (RP_Filter)
RHEL 9 drops packets between nodes if the return path isn't "strictly" identical. We moved this to loose mode:

Bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0
# Persist for reboot:
echo "net.ipv4.conf.all.rp_filter=0" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
### 3. Storage Layer: Longhorn Troubleshooting
We encountered "502 Bad Gateway" and "Connection Refused" errors during volume attachment.

Issue 1 (Webhook Loop): The Longhorn Admission Webhook was failing to validate requests.

Fix: Reset the webhooks to clear the stale API state.

Issue 2 (Identity Mismatch): Pods couldn't find servera.example.com.

Fix: Synchronized hostnamectl with the Kubernetes node name and updated /etc/hosts.

Issue 3 (No Route to Host): Node B could not reach the manager on Node A.

Fix: Resolved via the firewalld trusted zones (Step 2A).

### 4. FreePBX Deployment
The deployment uses a Persistent Volume Claim (PVC) to ensure your phone settings survive a pod crash.

Image: tiredofit/freepbx

Volume Size: 10GB (Longhorn Block Storage)

First Boot: Requires ~15-30 minutes for the container to "fetch" the FreePBX framework and build the MariaDB database.

### 5. Initial PBX Configuration
Web GUI: http://[NODE_IP]/admin

First Action: Created Admin User and bypassed the internal FreePBX Firewall (to prevent conflicts with RHEL's firewalld).

Extensions: Created Extension 101 (PJSIP) for mobile/desktop softphone testing.

Echo Test: Verified audio path via *43.

### 6. Post-Install Maintenance
Handy commands used during this session:

Force Update: kubectl rollout restart deployment [name]

Permissions Fix: kubectl exec -it [pod-name] -- fwconsole chown

Real-time Call Logs: kubectl exec -it [pod-name] -- asterisk -rvvvvv

### Next Steps
[ ] Configure SIP Trunk for external calling.

[ ] Verify Longhorn snapshot schedule for backups.
