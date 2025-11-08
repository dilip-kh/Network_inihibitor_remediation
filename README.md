# Network Inhibitor Remediation - Interface Renaming

Ansible playbook to remediate network inhibitor issues for RHEL 7 to 8 upgrades by renaming network interfaces from legacy `eth*` naming to predictable `en*` naming scheme.

## Overview

During RHEL 7 to 8 in-place upgrades, network interface naming conventions change from the traditional `eth0`, `eth1` format to predictable names like `eno1`, `ens3`, etc. This playbook automates the renaming process to prevent network connectivity issues post-upgrade.

## What This Playbook Does

The playbook performs comprehensive network interface renaming across multiple configuration layers:

1. **udev Rules** - Updates persistent network device rules with new interface names
2. **Interface Configuration Files** - Renames `ifcfg-eth*` files to `ifcfg-en*`
3. **Device Configuration** - Updates `NAME` and `DEVICE` parameters in network scripts
4. **Route Files** - Renames and updates `route-eth*` files to use new interface names
5. **TripleO Configuration** - Updates OpenStack TripleO-specific configurations (if present)
   - Undercloud configuration
   - os-net-config JSON files

## Features

- ✅ Automatic detection of all `eth*` interfaces
- ✅ Creates backups of original configuration files (`.bak` extension)
- ✅ Updates udev persistent network rules with MAC addresses
- ✅ Comprehensive search and replace across all network configuration files
- ✅ Support for OpenStack TripleO deployments
- ✅ Handles both simple and complex network setups (routes, VLANs, bonds)
- ✅ Tag-based execution for granular control

## Prerequisites

- RHEL/CentOS 7 or 8 system
- Ansible 2.9 or higher
- Root or sudo access
- Network interfaces using `eth*` naming convention

## Variables

| Variable          | Default                          | Description                               |
| ----------------- | -------------------------------- | ----------------------------------------- |
| `src_prefix`      | `eth`                            | Source interface prefix to rename from    |
| `dst_prefix`      | `en`                             | Destination interface prefix to rename to |
| `osnet_conf`      | `/etc/os-net-config/config.json` | OpenStack os-net-config file path         |
| `undercloud_conf` | `~/undercloud.conf`              | OpenStack undercloud configuration path   |

## Usage

### Basic Execution

```bash
# Run against localhost
ansible-playbook -i localhost, -c local interface_update.yml

# Run against remote hosts
ansible-playbook -i inventory.ini interface_update.yml

# Run with custom prefixes
ansible-playbook -i inventory.ini interface_update.yml -e "src_prefix=eth dst_prefix=ens"
```

### Inventory Example

Create an `inventory.ini` file:

```ini
[rhel_servers]
server1.example.com
server2.example.com
server3.example.com

[rhel_servers:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### Available Tags

Execute specific sections of the playbook using tags:

- `udev` - Update only udev persistent network rules
- `ifcfg` - Rename and update interface configuration files
- `routes` - Rename and update route files
- `tripleo` - Update OpenStack TripleO configurations

### Examples with Tags

```bash
# Only update udev rules
ansible-playbook -i inventory.ini interface_update.yml --tags udev

# Update interface configs and routes
ansible-playbook -i inventory.ini interface_update.yml --tags ifcfg,routes

# Full run excluding TripleO configs
ansible-playbook -i inventory.ini interface_update.yml --skip-tags tripleo
```

## Detailed Process

### 1. Interface Detection

The playbook automatically discovers all interfaces matching the source prefix (`eth*`) using Ansible facts.

### 2. udev Rules Creation

Creates persistent udev rules based on MAC addresses:

```
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="XX:XX:XX:XX:XX:XX", NAME="en0"
```

### 3. Interface Configuration Files

- Copies `ifcfg-eth0` → `ifcfg-en0`
- Updates `NAME=en0` and `DEVICE=en0` parameters
- Backs up original files to `.bak`
- Removes old `eth*` configuration files

### 4. Route Files

- Renames `route-eth0` → `route-en0`
- Updates interface references within route files
- Backs up and removes old route files

### 5. Final Cleanup

Performs a comprehensive regex search across all network configuration files to catch any remaining references to old interface names.

### 6. TripleO Integration (Optional)

Updates OpenStack-specific configuration files if they exist.

## File Modifications

### Before Execution

```
/etc/sysconfig/network-scripts/
├── ifcfg-eth0
├── ifcfg-eth1
├── route-eth0
└── route-eth1

/etc/udev/rules.d/
└── (no persistent rules)
```

### After Execution

```
/etc/sysconfig/network-scripts/
├── ifcfg-en0
├── ifcfg-en1
├── ifcfg-eth0.bak
├── ifcfg-eth1.bak
├── route-en0
├── route-en1
├── route-eth0.bak
└── route-eth1.bak

/etc/udev/rules.d/
└── 70-rhosp-persistent-net.rules
```

## Post-Execution Steps

1. **Verify Configuration**

   ```bash
   # Check interface files
   ls -la /etc/sysconfig/network-scripts/ifcfg-en*

   # Verify udev rules
   cat /etc/udev/rules.d/70-rhosp-persistent-net.rules
   ```

2. **Test Configuration (Before Reboot)**

   ```bash
   # Review what interfaces will be renamed
   ip link show | grep eth
   ```

3. **Reboot System**

   ```bash
   systemctl reboot
   ```

4. **Post-Reboot Verification**

   ```bash
   # Check new interface names
   ip link show
   nmcli device status

   # Verify network connectivity
   ping -c 4 8.8.8.8
   ip route show
   ```

## Rollback Procedure

If issues occur after reboot:

1. **Boot into rescue mode or single-user mode**

2. **Restore from backups**

   ```bash
   cd /etc/sysconfig/network-scripts/

   # Restore interface configs
   for file in ifcfg-*.bak; do
       mv "$file" "${file%.bak}"
   done

   # Restore route files
   for file in route-*.bak; do
       mv "$file" "${file%.bak}"
   done

   # Remove new interface files
   rm -f ifcfg-en* route-en*

   # Remove udev rules
   rm -f /etc/udev/rules.d/70-rhosp-persistent-net.rules
   ```

3. **Reboot**
   ```bash
   systemctl reboot
   ```

## Common Scenarios

### Standard RHEL Server

Renames `eth0`, `eth1` to `en0`, `en1`:

```bash
ansible-playbook -i inventory.ini interface_update.yml
```

### OpenStack Deployment

Includes TripleO configuration updates:

```bash
ansible-playbook -i inventory.ini interface_update.yml --tags all
```

### Test Run (Check Mode)

Preview changes without applying them:

```bash
ansible-playbook -i inventory.ini interface_update.yml --check --diff
```

## Compatibility

- **RHEL/CentOS 7.x** - Primary target for pre-upgrade remediation
- **RHEL/CentOS 8.x** - Can be used for post-upgrade cleanup
- **OpenStack deployments** - Full support for TripleO configurations

## Important Notes

⚠️ **Network Connectivity**: This playbook requires a reboot to apply udev rules. Ensure you have console access.

⚠️ **Backup Strategy**: Original files are backed up with `.bak` extension. Keep these until you verify the new configuration works.

⚠️ **Bond/Bridge/VLAN**: The playbook handles complex network configurations including bonds, bridges, and VLANs.

⚠️ **MAC Address Binding**: Udev rules use MAC addresses for persistent naming. Ensure MAC addresses won't change.

## Troubleshooting

### Issue: Interfaces not renamed after reboot

**Solution**:

```bash
# Regenerate initramfs with new udev rules
dracut -f
reboot
```

### Issue: Network not coming up after reboot

**Solution**:

```bash
# Check interface status
nmcli device status
ip link show

# Manually bring up interface
nmcli connection up en0
```

### Issue: Can't SSH after reboot

**Solution**: Use console access and verify:

```bash
# Check if NetworkManager is running
systemctl status NetworkManager

# Check interface configuration
cat /etc/sysconfig/network-scripts/ifcfg-en0

# Restart networking
systemctl restart NetworkManager
```

## Integration with RHEL Upgrade

This playbook should be run **before** executing the Leapp upgrade:

```bash
# 1. Run this playbook
ansible-playbook -i inventory.ini interface_update.yml

# 2. Reboot and verify
systemctl reboot

# 3. Check network post-reboot
nmcli device status

# 4. Proceed with Leapp upgrade
leapp preupgrade
leapp upgrade
```

## License

MIT License

## Contributing

Contributions are welcome! Please test thoroughly before submitting changes.

## References

- [RHEL 8 Network Interface Naming](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/)
- [Predictable Network Interface Names](https://www.freedesktop.org/wiki/Software/systemd/PredictableNetworkInterfaceNames/)
- [RHEL 7 to 8 Upgrade Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/upgrading_from_rhel_7_to_rhel_8/)
