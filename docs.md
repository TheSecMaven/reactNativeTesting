Excellent — this question breaks into two logical parts:
(1) **Operational discovery:** how to determine replication status and provisioning parameters from an existing host + LDEV pair, and
(2) **Data modeling:** how to persist those attributes and relationships in your own schema or data lake.

Below is a compact, expert-level explanation written for engineers already familiar with Hitachi Ops Center (Admin & CMREST) and Hitachi replication technologies (GAD / TrueCopy / ShadowImage).

---

# 🧭 Part 1 — Determine replication and provisioning context

### Step 1 — Identify host ↔ LDEV link

Use **Admin API** to confirm presentation:

```http
GET /v1/compute/servers/attached-volumes/?q=serverId:{serverId}
```

→ returns `volumeId`, `storageSystemId`, `lun`, `storagePortId`, `hostGroupId`.
Then:

```http
GET /v1/storage-systems/{sid}/volumes/{volumeId}
```

→ includes `replicationType`, `replicationGroupId`, `isPrimary`, and `pairedStorageSystemId`.

---

### Step 2 — Check replication status

**Admin API (volume attributes)**

| Key                     | Meaning                                                            |
| ----------------------- | ------------------------------------------------------------------ |
| `replicationType`       | `None`, `TrueCopy`, `UniversalReplicator`, `ShadowImage`, or `GAD` |
| `replicationRole`       | `Primary` / `Secondary` / `Duplex`                                 |
| `replicationGroupId`    | group membership across LDEVs                                      |
| `pairedStorageSystemId` | target array ID                                                    |
| `pairVolumeId`          | partner LDEV on remote array                                       |

**CMREST (Config Mgr Search):**

```http
POST /ConfigurationManager/v1/objects/replication
```

Search `replicationType:truecopy` or `gad` for deeper state (`pairStatus`, `consistencyGroupId`, `journalId`).

**CCI verification:**

```bash
raidcom get pair -ldev_id 00:1234
```

→ Shows pair type, role, status, and secondary LDEV.

---

### Step 3 — If server is *not* replicated

Provisioning requires these data elements:

| Category          | Example Attributes                                            | Source                                 |
| ----------------- | ------------------------------------------------------------- | -------------------------------------- |
| **Array context** | `storageSystemId`, model, serial                              | Admin `/storage-systems`               |
| **Pool / Parity** | `poolId`, `parityGroupId`, RAID level, capacity               | Admin `/pools`, `/parity-groups`       |
| **Front-end**     | `storagePortId`, `hostGroupId`, `hostMode`, `hostModeOptions` | Admin `/storage-ports`, `/host-groups` |
| **Presentation**  | `lunId`, `pathCount`                                          | Admin `attached-volumes`               |
| **Server meta**   | `serverId`, `osType`, `wwpn`/`iqn`                            | Admin `/compute/servers`               |

Use these when calling `POST /v1/storage-systems/{sid}/volumes` and then mapping via `edit-lun-paths`.

---

### Step 4 — If server is replicated

Determine replication configuration first:

| Replication Type               | Key APIs / Attributes                                               | Typical Partner Info                                      |
| ------------------------------ | ------------------------------------------------------------------- | --------------------------------------------------------- |
| **TrueCopy**                   | CMREST `replicationType:truecopy`; Admin `replicationType=TRUECOPY` | `primaryLdevId`, `secondaryStorageSystemId`, `pairStatus` |
| **Universal Replicator (UR)**  | CMREST `replicationType:universalreplicator`                        | Adds `journalId`, `muNumber`, `deltaTime`                 |
| **ShadowImage (SI)**           | CMREST `replicationType:shadowimage`                                | `copyGroupName`, `splitStatus`                            |
| **Global-Active Device (GAD)** | CMREST `replicationType:gad`                                        | `clusterId`, `pathGroupId`, `mirrorStatus`                |

**Provisioning for replicated volumes** adds these data elements:

* **Primary array info:** `storageSystemId`, `poolId`, `ldevId`
* **Secondary array info:** `pairedStorageSystemId`, `pairedPoolId`
* **Replication group/journal:** `replicationGroupId`, `journalId`, `consistencyGroupId`
* **Replication mode:** sync/async, `pairStatus`, `role`
* **Copy method:** `replicationType`
* **Path policy:** same number of paths/host groups on both arrays
* **CCI verification:** `raidcom get pair -ldev_id {id}` (confirm `PAIR` state = `PAIR/PAIR-SYNC`)

If provisioning new storage for replicated servers, ensure:

1. Create primary LDEVs on both arrays (matching capacity & pool type).
2. Register in same replication group.
3. Create LU paths on both sides.
4. Start pair creation via Admin API `POST /v1/replications` or via CCI `paircreate`.

---

# 🧩 Part 2 — Data-modeling attributes (#DATA)

To persist topology and configuration, model each entity and its key attributes.

| Entity            | Key Attributes                                                                                         | Description             |
| ----------------- | ------------------------------------------------------------------------------------------------------ | ----------------------- |
| **StorageSystem** | `storageSystemId`, `model`, `serialNumber`, `microcodeVersion`                                         | Top-level array         |
| **Pool**          | `poolId`, `raidType`, `totalCapacity`, `availableCapacity`                                             | Capacity source         |
| **ParityGroup**   | `parityGroupId`, `raidLevel`, `driveType`, `speed`                                                     | RAID backend            |
| **Volume (LDEV)** | `ldevId`, `sizeMB`, `poolId`, `parityGroupId`, `replicationType`, `replicationGroupId`, `pairVolumeId` | Logical volume metadata |
| **Port**          | `storagePortId`, `portMode`, `wwn`, `protocol`                                                         | Front-end port          |
| **HostGroup**     | `hostGroupId`, `portId`, `hostMode`, `hostModeOptions`, `initiators[]`                                 | Presentation group      |
| **LUN Path**      | `lunId`, `hostGroupId`, `ldevId`, `pathStatus`                                                         | Join object             |
| **Server**        | `serverId`, `hostname`, `osType`, `clusterGroup`, `isReplicated`                                       | Host metadata           |
| **Replication**   | `replicationType`, `replicationGroupId`, `journalId`, `pairStatus`, `primaryLdevId`, `secondaryLdevId` | Pairing configuration   |

These attributes allow you to build a normalized data model that matches Ops Center’s internal schema and supports both replicated and non-replicated provisioning workflows.

---

## 📘 References

* [Ops Center Admin API Guide](https://knowledge.hitachivantara.com/Documents/Management_Software/Ops_Center/API)
* [Ops Center Configuration Manager REST API Guide](https://knowledge.hitachivantara.com/Documents/Management_Software/Ops_Center/Configuration_Manager)
* [Ops Center Analyzer Detail View Guide (MQL)](https://knowledge.hitachivantara.com/)
* [Command Control Interface Manual](https://knowledge.hitachivantara.com/Documents/Management_Software/Command_Control_Interface)

---

### ✅ Summary

* Use **Admin API** for integrated host/volume discovery.
* Use **CMREST** for replication and low-level array state.
* Capture key attributes (pool, parity group, replication group, etc.) into your data model for consistent provisioning and reporting.
