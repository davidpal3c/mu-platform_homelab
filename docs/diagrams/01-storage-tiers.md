# 01 — Storage tiers

**Reflects the built state of `mu-node-01`.** Narrative: [../storage.md](../storage.md).

```mermaid
flowchart LR
    subgraph Devices["Physical devices"]
        SDA["sda<br/>Intel SATA SSD ~180 GB"]
        SDC["sdc<br/>Intel SATA SSD ~180 GB"]
        NVME1["nvme1n1<br/>WD SN530 ~238 GB"]
        NVME0["nvme0n1<br/>WD SN530 ~477 GB"]
        SDB["sdb<br/>SATA HDD ~931 GB"]
    end

    subgraph Boot["Boot tier — mdadm RAID1"]
        MD0["md0"]
        MD1["md1"]
        MD2["md2"]
        ESP["ESP: sda1 primary<br/>sdc1 synced secondary"]
    end

    SDA --> MD0
    SDA --> MD1
    SDA --> MD2
    SDC --> MD0
    SDC --> MD1
    SDC --> MD2
    SDA --> ESP
    SDC --> ESP

    MD0 --> M_BOOT["/boot"]
    MD1 --> M_ROOT["/"]
    MD2 --> M_VAR["/var"]

    NVME1 --> M_PLAT["/platform<br/>ext4 · UUID · nofail"]
    NVME0 --> M_DATA["/data<br/>ext4 · UUID · nofail"]
    SDB --> M_BKP["/backups<br/>ext4 · UUID · nofail"]

    subgraph Binds["Bind mounts — bind,nofail"]
        B1["/var/lib/rancher"]
        B2["/var/lib/kubelet"]
        B3["/var/lib/containerd"]
        B4["/var/lib/prometheus"]
        B5["/var/lib/loki"]
    end

    M_PLAT --> B1
    M_PLAT --> B2
    M_PLAT --> B3
    M_PLAT --> B4
    M_PLAT --> B5

    subgraph DataDirs["Database tier directories"]
        D1["/data/postgres"]
        D2["/data/postgres_wal"]
        D3["/data/redis"]
    end

    M_DATA --> D1
    M_DATA --> D2
    M_DATA --> D3
```

**What the diagram is showing:** every high-churn path on the node resolves to `/platform`, and every latency-sensitive path resolves to `/data`. Neither touches the boot RAID.

**Two facts that must not drift:** the *larger* NVMe is the database tier, and the backup device is `sdb`. `sdc` is the second RAID1 boot SSD.
