# Home Router Installation - Ansible Project

This Ansible project automates the setup and configuration of a private router on a Debian machine.

## Project Structure

```
.
├── ansible.cfg           # Ansible configuration
├── inventory.ini         # Inventory file for target hosts
├── common.yml           # Common operations playbook
├── roles/               # Custom roles directory
├── group_vars/          # Group variables
└── host_vars/           # Host-specific variables
```

## Prerequisites

- Ansible installed on your control machine
- SSH access to the target Debian router
- Root or sudo privileges on the target machine

## Setup

1. **Configure your inventory**: Edit `inventory.ini` and add your router's IP address or hostname:
   ```ini
   [router]
   router.local ansible_host=192.168.1.1 ansible_user=root
   ```

2. **Test connectivity**:
   ```bash
   ansible router -m ping
   ```

3. **Bootstrap the router** (First time only - if sudo is not installed):
   ```bash
   # Run this once as root to install sudo and set up the rkaper user
   ansible-playbook bootstrap.yml -u root
   ```

## Playbooks

### bootstrap.yml - Initial Bootstrap (Run Once)

This playbook should be run **once** on a fresh Debian installation to install basic requirements:
- Installs sudo
- Installs Python3 (required by Ansible)

**Must be run as root:**
```bash
ansible-playbook bootstrap.yml -u root
```

After running this, all other playbooks can use the `rkaper` user with sudo privileges.

### common.yml - Common Operations

This playbook handles basic system setup:
- Ensures the `rkaper` user exists with sudo group membership
- **Configures passwordless sudo for rkaper**
- Updates system packages
- Installs essential utilities
- Sets up SSH directory structure

**Run the playbook:**
```bash
ansible-playbook common.yml
```

**Run specific tags:**
```bash
# Only update packages
ansible-playbook common.yml --tags update

# Only create user
ansible-playbook common.yml --tags user

# Only install packages
ansible-playbook common.yml --tags packages
```

### network.yml - LAN Network Configuration

This playbook configures the router's local network interface with a static IP:
- Sets up static IP address (default: 192.168.2.2)
- Configures network interface settings
- Enables IP forwarding for routing
- Backs up existing network configuration

**Run the playbook:**
```bash
ansible-playbook network.yml
```

**Run specific tags:**
```bash
# Only check interface facts
ansible-playbook network.yml --tags facts

# Configure network without restarting
ansible-playbook network.yml --tags configure --skip-tags restart
```

**Warning:** The network playbook will restart networking services, which may temporarily disconnect your SSH session.

### wan.yml - WAN PPPoE Configuration

This playbook configures the WAN connection using PPPoE for KPN:
- Installs PPPoE and VLAN packages
- Configures VLAN interface (default: VLAN 6)
- Creates PPP peers configuration file
- Sets up PPPoE authentication

**Configuration in inventory.ini:**
```ini
wan_physical_interface=enp1s0
wan_vlan=6
ppp_provider=kpn
ppp_user=kpn@kpn.nl
```

**Run the playbook:**
```bash
ansible-playbook wan.yml
```

**Manual connection control:**
```bash
# Start PPPoE connection
pon kpn

# Stop PPPoE connection
poff kpn

# Check connection status
plog
```

**Note:** You'll need to set the PPP password in `/etc/ppp/pap-secrets` or `/etc/ppp/chap-secrets` manually if required by your ISP.

## Recommended Workflow

For a fresh Debian installation, run playbooks in this order:

```bash
# 1. Run common setup (updates system, installs packages, configures sudo)
ansible-playbook common.yml

# 2. Check network interfaces (to identify correct interface names)
ansible-playbook check_interfaces.yml

# 3. Configure LAN network (update local_interface in inventory.ini first if needed)
ansible-playbook network.yml

# 4. Configure WAN PPPoE connection (KPN)
ansible-playbook wan.yml

# 5. After network changes, update inventory.ini to use the new IP (192.168.2.2)
# Then start the PPPoE connection
ssh rkaper@192.168.2.2
sudo pon kpn
```

## Usage Examples

```bash
# Run common setup
ansible-playbook common.yml

# Check what would change (dry run)
ansible-playbook common.yml --check

# Run with verbose output
ansible-playbook common.yml -v
```

## Security Notes

- The playbook sets up passwordless sudo for the `rkaper` user
- Make sure to secure SSH keys properly
- Consider changing the default SSH port after initial setup

## Next Steps

Consider adding more playbooks for:
- Firewall rules (iptables/nftables) with NAT
- DHCP server setup (dnsmasq/isc-dhcp-server)
- DNS caching/resolver (unbound/dnsmasq)
- VPN setup (WireGuard/OpenVPN)
- Monitoring tools (Prometheus/Grafana)
- QoS (Traffic shaping)

