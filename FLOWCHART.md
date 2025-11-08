# Network Interface Renaming Playbook - Execution Flowchart

This document provides a visual flowchart of the `interface_update.yml` playbook execution flow.

## High-Level Process Flow

```
┌─────────────────────────────────────────┐
│   START: Ansible Playbook Execution    │
│   Target: All hosts with become: true  │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│      Gather Facts & Load Variables      │
│  - ansible_interfaces                   │
│  - src_prefix: "eth"                    │
│  - dst_prefix: "en"                     │
│  - osnet_conf path                      │
│  - undercloud_conf path                 │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│     Identify Source Interfaces          │
│  Filter: ansible_interfaces matching    │
│          src_prefix pattern (eth*)      │
│  Output: src_interfaces list            │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│       Display Interface List            │
│  Debug: Show all eth* interfaces found  │
└────────────────┬────────────────────────┘
                 │
                 ▼
        ┌────────┴────────┐
        │  Tag Selection  │
        └────────┬────────┘
                 │
    ┌────────────┼────────────┬────────────┐
    │            │            │            │
    ▼            ▼            ▼            ▼
┌────────┐  ┌────────┐  ┌─────────┐  ┌─────────┐
│  udev  │  │ ifcfg  │  │ routes  │  │ tripleo │
│  tag   │  │  tag   │  │   tag   │  │   tag   │
└───┬────┘  └───┬────┘  └────┬────┘  └────┬────┘
    │           │            │            │
    │           │            │            │
    └───────────┴────────────┴────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│              END PLAYBOOK               │
│      All Tasks Completed Successfully   │
└─────────────────────────────────────────┘
```

---

## Detailed Task Flow by Tag

### 1. UDEV Rules Update (Tag: `udev`)

```
┌─────────────────────────────────────────┐
│    START: Update udev Rules Task       │
│    Tag: udev                            │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   Loop: For each eth* interface        │
│   (excluding VLAN interfaces)           │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   Get MAC Address from ansible_facts    │
│   - Prefer: perm_macaddress             │
│   - Fallback: macaddress                │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   Create/Update udev Rule Line:         │
│   SUBSYSTEM=="net", ACTION=="add",      │
│   DRIVERS=="?*",                        │
│   ATTR{address}=="MAC",                 │
│   NAME=="enX"                           │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   Write to:                             │
│   /etc/udev/rules.d/                    │
│   70-rhosp-persistent-net.rules         │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   COMPLETE: udev Rules Updated          │
└─────────────────────────────────────────┘
```

---

### 2. Interface Configuration Files (Tag: `ifcfg`)

```
┌─────────────────────────────────────────┐
│   START: Rename ifcfg Files Block      │
│   Tag: ifcfg                            │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 1: Check File Existence          │
│   For each eth* interface:              │
│   Check: /etc/sysconfig/network-scripts/│
│          ifcfg-ethX exists?             │
└────────────────┬────────────────────────┘
                 │
                 ▼
        ┌────────┴────────┐
        │  File Exists?   │
        └────────┬────────┘
                 │
        ┌────────┴────────┐
        │                 │
       Yes               No
        │                 │
        ▼                 ▼
┌──────────────┐    ┌──────────────┐
│  Continue    │    │  Skip This   │
│  Processing  │    │  Interface   │
└──────┬───────┘    └──────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│   TASK 2: Copy ifcfg File               │
│   Copy: ifcfg-ethX → ifcfg-enX          │
│   remote_src: true                      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 3: Edit NAME Parameter           │
│   Update in ifcfg-enX:                  │
│   NAME=enX                              │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 4: Edit DEVICE Parameter         │
│   Update in ifcfg-enX:                  │
│   DEVICE=enX                            │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 5: Backup Original File          │
│   Copy: ifcfg-ethX → ifcfg-ethX.bak     │
│   remote_src: true                      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 6: Remove Original File          │
│   Delete: ifcfg-ethX                    │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 7: Final Regex Cleanup           │
│   Find all ifcfg-* files (exclude .bak) │
│   Replace any remaining ethX refs       │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   COMPLETE: ifcfg Files Renamed         │
└─────────────────────────────────────────┘
```

---

### 3. Route Configuration Files (Tag: `routes`)

```
┌─────────────────────────────────────────┐
│   START: Rename Route Files Block      │
│   Tag: routes                           │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 1: Check Route File Existence    │
│   For each eth* interface:              │
│   Check: /etc/sysconfig/network-scripts/│
│          route-ethX exists?             │
└────────────────┬────────────────────────┘
                 │
                 ▼
        ┌────────┴────────┐
        │  File Exists?   │
        └────────┬────────┘
                 │
        ┌────────┴────────┐
        │                 │
       Yes               No
        │                 │
        ▼                 ▼
┌──────────────┐    ┌──────────────┐
│  Continue    │    │  Skip This   │
│  Processing  │    │  Interface   │
└──────┬───────┘    └──────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│   TASK 2: Copy Route File               │
│   Copy: route-ethX → route-enX          │
│   remote_src: true                      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 3: Update Content                │
│   In route-enX file:                    │
│   Replace all "eth" → "en"              │
│   (IP command arguments format)         │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 4: Backup Original File          │
│   Copy: route-ethX → route-ethX.bak     │
│   remote_src: true                      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   TASK 5: Remove Original File          │
│   Delete: route-ethX                    │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   COMPLETE: Route Files Renamed         │
└─────────────────────────────────────────┘
```

---

### 4. TripleO Configuration Update (Tag: `tripleo`)

```
┌─────────────────────────────────────────┐
│   START: TripleO Config Update Block    │
│   Tag: tripleo                          │
└────────────────┬────────────────────────┘
                 │
                 ├──────────────────┐
                 │                  │
                 ▼                  ▼
┌────────────────────────┐  ┌────────────────────────┐
│  Undercloud Config     │  │  os-net-config         │
│  Update Path           │  │  Update Path           │
└───────────┬────────────┘  └────────┬───────────────┘
            │                        │
            ▼                        ▼
┌────────────────────────┐  ┌────────────────────────┐
│  Check File Exists:    │  │  Check File Exists:    │
│  ~/undercloud.conf     │  │  /etc/os-net-config/   │
│                        │  │  config.json           │
└───────────┬────────────┘  └────────┬───────────────┘
            │                        │
    ┌───────┴────────┐      ┌────────┴───────┐
    │  File Exists?  │      │  File Exists?  │
    └───────┬────────┘      └────────┬───────┘
            │                        │
    ┌───────┴───────┐        ┌───────┴───────┐
   Yes             No       Yes             No
    │               │        │               │
    ▼               ▼        ▼               ▼
┌─────────┐   ┌────────┐  ┌─────────┐   ┌────────┐
│ Update  │   │  Skip  │  │ Update  │   │  Skip  │
│  File   │   │        │  │  File   │   │        │
└────┬────┘   └────────┘  └────┬────┘   └────────┘
     │                          │
     ▼                          ▼
┌─────────────────┐    ┌─────────────────┐
│ Replace ethX    │    │ Replace ethX    │
│ with enX in     │    │ with enX in     │
│ undercloud.conf │    │ config.json     │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│   COMPLETE: TripleO Configs Updated     │
└─────────────────────────────────────────┘
```

---

## Complete Execution Sequence

### Timeline View

```
TIME
 │
 ├─► [T0] Playbook Start
 │    └─► Gather facts
 │    └─► Load variables
 │    └─► Identify eth* interfaces
 │
 ├─► [T1] udev Tag Tasks
 │    └─► Create udev rules for each interface
 │    └─► Write to 70-rhosp-persistent-net.rules
 │
 ├─► [T2] ifcfg Tag Tasks
 │    └─► Check ifcfg-eth* files exist
 │    └─► Copy to ifcfg-en* files
 │    └─► Update NAME parameter
 │    └─► Update DEVICE parameter
 │    └─► Backup original files (.bak)
 │    └─► Remove original files
 │    └─► Final regex cleanup
 │
 ├─► [T3] routes Tag Tasks
 │    └─► Check route-eth* files exist
 │    └─► Copy to route-en* files
 │    └─► Update interface references in content
 │    └─► Backup original files (.bak)
 │    └─► Remove original files
 │
 ├─► [T4] tripleo Tag Tasks
 │    └─► Check undercloud.conf exists
 │    └─► Update interface names in undercloud.conf
 │    └─► Check config.json exists
 │    └─► Update interface names in config.json
 │
 └─► [T5] Playbook Complete
      └─► Summary report
      └─► Reboot required for changes to take effect
```

---

## Decision Matrix

### File Processing Logic

```
For each interface in src_interfaces:
│
├─► Is it a VLAN interface (contains '.')?
│   ├─► YES: Skip udev rule creation
│   └─► NO: Create udev rule
│
├─► Does ifcfg-ethX file exist?
│   ├─► YES: Process file (copy, edit, backup, remove)
│   └─► NO: Skip interface config processing
│
├─► Does route-ethX file exist?
│   ├─► YES: Process file (copy, edit, backup, remove)
│   └─► NO: Skip route processing
│
└─► TripleO files exist?
    ├─► undercloud.conf exists?
    │   ├─► YES: Update interface references
    │   └─► NO: Skip
    │
    └─► config.json exists?
        ├─► YES: Update interface references
        └─► NO: Skip
```

---

## Data Flow Diagram

```
┌──────────────────────────────────────────────────────┐
│              INPUT: System Facts                     │
│  - ansible_interfaces: [eth0, eth1, eth2, lo, ...]  │
│  - ansible_facts[ethX][macaddress]                   │
│  - ansible_facts[ethX][perm_macaddress]              │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│            PROCESSING: Filter & Transform            │
│  - Filter: Select only eth* interfaces              │
│  - Transform: eth0 → en0, eth1 → en1, etc.          │
│  - Exclude: VLAN interfaces (eth0.100)              │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│              OUTPUT: Modified Files                  │
│  - /etc/udev/rules.d/70-rhosp-persistent-net.rules  │
│  - /etc/sysconfig/network-scripts/ifcfg-en*         │
│  - /etc/sysconfig/network-scripts/route-en*         │
│  - ~/undercloud.conf (if exists)                    │
│  - /etc/os-net-config/config.json (if exists)       │
│                                                      │
│              BACKUP: Original Files                  │
│  - ifcfg-eth*.bak                                   │
│  - route-eth*.bak                                   │
└──────────────────────────────────────────────────────┘
```

---

## Error Handling Flow

```
┌────────────────────────────┐
│   Task Execution Start     │
└──────────┬─────────────────┘
           │
           ▼
    ┌──────────────┐
    │  stat check  │
    │  file exists │
    └──────┬───────┘
           │
    ┌──────┴──────┐
    │   Exists?   │
    └──────┬──────┘
           │
    ┌──────┴──────┐
   YES           NO
    │             │
    ▼             ▼
┌─────────┐  ┌──────────┐
│Process  │  │ Skip     │
│File     │  │ (when:   │
│         │  │ condition│
└────┬────┘  │ not met) │
     │       └──────────┘
     ▼
┌─────────────────────┐
│  File Operation     │
│  (copy/edit/remove) │
└────┬────────────────┘
     │
     ▼
┌─────────────────────┐
│   Success or Fail?  │
└────┬────────────────┘
     │
┌────┴────┐
│         │
Success  Fail
│         │
▼         ▼
┌─────┐ ┌──────────────┐
│Next │ │ Playbook     │
│Task │ │ Stops (fatal)│
└─────┘ └──────────────┘
```

---

## Post-Execution Workflow

```
┌─────────────────────────────────────────┐
│   Playbook Execution Complete           │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   MANUAL STEP 1: Verify Files           │
│   - Check new ifcfg-en* files           │
│   - Check udev rules file               │
│   - Verify backups exist (.bak)         │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   MANUAL STEP 2: Reboot System          │
│   - systemctl reboot                    │
│   - Ensure console access available     │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   MANUAL STEP 3: Post-Reboot Verify     │
│   - Check interface names: ip link      │
│   - Check connectivity: ping test       │
│   - Check NetworkManager: nmcli         │
└────────────────┬────────────────────────┘
                 │
         ┌───────┴────────┐
         │   Success?     │
         └───────┬────────┘
                 │
         ┌───────┴───────┐
        YES             NO
         │               │
         ▼               ▼
┌─────────────┐  ┌─────────────────┐
│ Proceed to  │  │   Rollback      │
│ RHEL 8      │  │   Procedure     │
│ Upgrade     │  │   (restore .bak)│
└─────────────┘  └─────────────────┘
```

---

## Summary Statistics

| **Metric**             | **Value**                                     |
| ---------------------- | --------------------------------------------- |
| Total Tags             | 4 (udev, ifcfg, routes, tripleo)              |
| Total Task Blocks      | 4 major blocks                                |
| Total Individual Tasks | ~20 tasks                                     |
| Files Modified         | ifcfg-_, route-_, udev rules, TripleO configs |
| Backup Files Created   | 2 per interface (ifcfg + route)               |
| Required Privilege     | root/sudo                                     |
| Idempotent             | Partially (can be run multiple times)         |
| Reboot Required        | Yes (for udev rules to take effect)           |

---

## Legend

```
┌─────────┐
│  Box    │  = Process/Task
└─────────┘

    │
    ▼        = Sequential Flow

┌────┴────┐
│         │  = Decision Point
YES      NO

[Tag]      = Ansible Tag

MANUAL     = Requires manual intervention
```
