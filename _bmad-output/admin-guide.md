# Slurm 管理員指南

> 產生日期：2025-12-17 | 適用對象：系統管理員

本指南提供 Slurm 叢集的安裝、配置和維護說明。

---

## 目錄

1. [架構概覽](#架構概覽)
2. [安裝與設定](#安裝與設定)
3. [核心配置](#核心配置)
4. [節點管理](#節點管理)
5. [分割區管理](#分割區管理)
6. [帳戶與記帳](#帳戶與記帳)
7. [QoS 管理](#qos-管理)
8. [排程與優先級](#排程與優先級)
9. [監控與維護](#監控與維護)
10. [故障排除](#故障排除)
11. [安全設定](#安全設定)
12. [高可用性](#高可用性)

---

## 架構概覽

### 系統元件

```
使用者 --> CLI/REST API --> slurmctld（控制器）
                                 |
                           slurmd（節點）
                                 |
                         slurmstepd（任務）
                                 |
                           slurmdbd --> MySQL
```

### 守護程式

| 守護程式 | 連接埠 | 用途 |
|---------|--------|------|
| slurmctld | 6817 | 中央控制器，排程與資源管理 |
| slurmd | 6818 | 節點守護程式，本機作業執行 |
| slurmdbd | 6819 | 資料庫守護程式，記帳服務 |
| slurmrestd | 6820 | REST API 介面 |

### 設定檔

| 檔案 | 用途 |
|------|------|
| `slurm.conf` | 主要配置檔 |
| `slurmdbd.conf` | 資料庫守護程式配置 |
| `cgroup.conf` | Cgroup 設定 |
| `gres.conf` | 通用資源（GPU 等）|
| `topology.conf` | 網路拓樸 |

---

## 安裝與設定

### 前置需求

```bash
# RHEL/CentOS
yum install munge munge-devel mariadb-server mariadb-devel \
    readline-devel openssl-devel pam-devel

# Ubuntu/Debian
apt install munge libmunge-dev mariadb-server libmariadb-dev \
    libreadline-dev libssl-dev libpam0g-dev
```

### 安裝 Slurm

**從原始碼：**
```bash
./configure --prefix=/usr/local --sysconfdir=/etc/slurm \
            --with-munge --enable-pam
make -j$(nproc)
make install
```

**RPM 建置：**
```bash
rpmbuild -ta slurm-*.tar.bz2 --with mysql --with slurmrestd
```

### MUNGE 設定

所有節點都需要 MUNGE 驗證：

```bash
# 在主節點產生金鑰
create-munge-key

# 複製到所有節點
scp /etc/munge/munge.key node01:/etc/munge/
scp /etc/munge/munge.key node02:/etc/munge/

# 設定權限
chown munge:munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key

# 啟動服務
systemctl enable --now munge
```

### 資料庫設定

```bash
# 建立資料庫
mysql -u root -p

CREATE DATABASE slurm_acct_db;
CREATE USER 'slurm'@'localhost' IDENTIFIED BY 'password';
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
```

---

## 核心配置

### slurm.conf 基本設定

```bash
# 叢集基本資訊
ClusterName=mycluster
SlurmctldHost=controller

# 驗證
AuthType=auth/munge
CryptoType=crypto/munge

# 排程
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# 記帳
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=controller
JobAcctGatherType=jobacct_gather/cgroup

# 日誌
SlurmctldDebug=info
SlurmdDebug=info
SlurmctldLogFile=/var/log/slurmctld.log
SlurmdLogFile=/var/log/slurmd.log

# 程序追蹤
ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup,task/affinity

# 節點定義
NodeName=node[01-10] CPUs=32 Boards=1 SocketsPerBoard=2 \
    CoresPerSocket=8 ThreadsPerCore=2 RealMemory=128000 State=UNKNOWN

# 分割區定義
PartitionName=compute Nodes=node[01-10] Default=YES MaxTime=7-00:00:00 State=UP
PartitionName=gpu Nodes=node[01-02] MaxTime=1-00:00:00 State=UP
```

### slurmdbd.conf 設定

```bash
# 記帳守護程式設定
AuthType=auth/munge
DbdHost=controller
DbdPort=6819
SlurmUser=slurm

# 資料庫連線
StorageType=accounting_storage/mysql
StorageHost=localhost
StoragePort=3306
StorageUser=slurm
StoragePass=password
StorageLoc=slurm_acct_db

# 日誌
DebugLevel=info
LogFile=/var/log/slurmdbd.log
PidFile=/var/run/slurmdbd.pid
```

### cgroup.conf 設定

```bash
# Cgroup 控制
CgroupAutomount=yes
ConstrainCores=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes
ConstrainDevices=yes
```

---

## 節點管理

### 節點狀態

```bash
# 檢視節點狀態
sinfo -N -l

# 檢視特定節點詳情
scontrol show node node01
```

### 節點維護

```bash
# 將節點設為維護模式（完成現有作業後下線）
scontrol update nodename=node01 state=drain reason="Maintenance"

# 立即下線節點
scontrol update nodename=node01 state=down reason="Emergency"

# 恢復節點
scontrol update nodename=node01 state=resume

# 重新啟動節點上的 slurmd
scontrol update nodename=node01 state=idle
```

### 動態節點

```bash
# 在 slurm.conf 中啟用
TreeWidth=65533  # 支援動態節點

# 動態新增節點
scontrol create nodename=newnode01 cpus=16 realmemory=64000

# 移除節點
scontrol delete nodename=newnode01
```

---

## 分割區管理

### 分割區設定

```bash
# slurm.conf 中的分割區定義
PartitionName=compute Nodes=node[01-08] Default=YES \
    MaxTime=7-00:00:00 DefaultTime=1:00:00 \
    MaxNodes=4 State=UP

PartitionName=gpu Nodes=node[09-10] \
    MaxTime=1-00:00:00 \
    Gres=gpu:4 State=UP

PartitionName=debug Nodes=node01 \
    MaxTime=00:30:00 \
    Priority=100 State=UP
```

### 動態管理

```bash
# 檢視分割區
scontrol show partition

# 修改分割區狀態
scontrol update partitionname=compute state=down

# 修改時間限制
scontrol update partitionname=compute maxtime=14-00:00:00
```

### 存取控制

```bash
# 限制特定使用者或帳戶
PartitionName=restricted Nodes=node[01-02] \
    AllowAccounts=research,engineering \
    State=UP

# 限制特定群組
PartitionName=staff Nodes=node[03-04] \
    AllowGroups=staff,admin \
    State=UP
```

---

## 帳戶與記帳

### 初始化記帳

```bash
# 啟動 slurmdbd
systemctl start slurmdbd

# 新增叢集
sacctmgr add cluster mycluster

# 新增根帳戶
sacctmgr add account root Description="Root Account"
```

### 帳戶階層

```bash
# 建立帳戶階層
sacctmgr add account engineering Description="Engineering Dept" \
    Organization="Engineering"

sacctmgr add account research Description="Research Dept" \
    Organization="Research"

# 建立子帳戶
sacctmgr add account ml Parent=engineering Description="ML Team"
sacctmgr add account physics Parent=research Description="Physics Group"
```

### 使用者管理

```bash
# 新增使用者
sacctmgr add user alice Account=engineering

# 新增使用者並設定限制
sacctmgr add user bob Account=research \
    MaxJobs=10 MaxSubmit=20 MaxWall=7-00:00:00

# 修改使用者
sacctmgr modify user alice set MaxJobs=20

# 檢視使用者
sacctmgr show user alice -s

# 移除使用者
sacctmgr delete user alice
```

### 資源限制

```bash
# 帳戶層級限制
sacctmgr modify account engineering set \
    GrpTRES=cpu=100,mem=500G \
    GrpJobs=50 \
    MaxWall=7-00:00:00

# 使用者層級限制
sacctmgr modify user alice set \
    MaxTRESPerJob=cpu=32,mem=128G \
    MaxJobs=5
```

### 使用報告

```bash
# 帳戶使用報告
sreport cluster AccountUtilizationByUser start=2025-01-01

# 使用者使用報告
sreport user TopUsage start=2025-01-01

# 叢集使用率
sreport cluster Utilization start=2025-01-01
```

---

## QoS 管理

### QoS 概念

QoS（服務品質）用於：
- 設定不同的優先級
- 控制搶佔行為
- 設定資源限制

### QoS 設定

```bash
# 建立 QoS
sacctmgr add qos normal Priority=50
sacctmgr add qos high Priority=100 Preempt=normal
sacctmgr add qos low Priority=10

# 設定 QoS 限制
sacctmgr modify qos high set \
    MaxTRESPerUser=cpu=64,mem=256G \
    MaxJobsPerUser=10 \
    MaxWall=1-00:00:00

# 指派 QoS 給帳戶
sacctmgr modify account engineering set QOS=normal,high

# 檢視 QoS
sacctmgr show qos
```

### 搶佔設定

```bash
# slurm.conf 中啟用搶佔
PreemptType=preempt/qos
PreemptMode=REQUEUE

# QoS 搶佔設定
sacctmgr modify qos high set Preempt=normal,low PreemptMode=REQUEUE
```

---

## 排程與優先級

### 多因素優先級

```bash
# slurm.conf 設定
PriorityType=priority/multifactor
PriorityWeightAge=1000
PriorityWeightFairshare=10000
PriorityWeightJobSize=500
PriorityWeightPartition=1000
PriorityWeightQOS=2000

# 優先級衰減
PriorityDecayHalfLife=7-0
PriorityMaxAge=14-0
```

### Backfill 排程

```bash
# slurm.conf 設定
SchedulerType=sched/backfill
SchedulerParameters=bf_max_job_test=1000,bf_resolution=300,bf_window=2880
```

### 公平共用

```bash
# 設定使用者份額
sacctmgr modify user alice set FairShare=100
sacctmgr modify user bob set FairShare=50

# 設定帳戶份額
sacctmgr modify account engineering set FairShare=60
sacctmgr modify account research set FairShare=40

# 檢視公平共用資訊
sshare -a
```

---

## 監控與維護

### 系統診斷

```bash
# 排程器診斷
sdiag

# 檢視配置
scontrol show config

# 檢視統計
scontrol show stats

# 檢視授權資訊
scontrol show lic
```

### 日誌管理

```bash
# 日誌位置
/var/log/slurmctld.log
/var/log/slurmd.log
/var/log/slurmdbd.log

# 動態調整除錯等級
scontrol setdebug debug2
scontrol setdebugflags +Backfill

# 恢復正常等級
scontrol setdebug info
```

### 設定重載

```bash
# 重新載入配置（不重啟服務）
scontrol reconfigure

# 重啟服務
systemctl restart slurmctld
systemctl restart slurmd
```

### 定期維護

```bash
# 清理舊作業記錄
sacctmgr delete job endtime<2024-01-01

# 資料庫維護
mysqlcheck -o slurm_acct_db
```

---

## 故障排除

### 常見問題

**slurmd 無法啟動：**
```bash
# 檢查配置
slurmd -C

# 檢查與控制器的連線
ping controller
telnet controller 6817

# 檢查 MUNGE
munge -n | unmunge
```

**作業無法排程：**
```bash
# 檢查分割區狀態
sinfo

# 檢查作業原因
squeue -u username -o "%i %j %T %r"

# 檢查排程器
sdiag | grep -A 20 "Backfill"
```

**節點狀態異常：**
```bash
# 檢查節點狀態
scontrol show node nodename

# 檢查節點上的 slurmd
ssh nodename "systemctl status slurmd"

# 強制更新節點狀態
scontrol update nodename=node01 state=idle
```

### 除錯模式

```bash
# 前景執行 slurmctld
slurmctld -D -vvv

# 前景執行 slurmd
slurmd -D -vvv
```

---

## 安全設定

### MUNGE 驗證

```bash
# 確保金鑰權限正確
chown munge:munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key

# 定期更換金鑰
create-munge-key --force
# 然後重新分發到所有節點
```

### JWT 驗證

```bash
# slurm.conf
AuthAltTypes=auth/jwt
AuthAltParameters=jwt_key=/etc/slurm/jwt_hs256.key

# 產生金鑰
dd if=/dev/urandom of=/etc/slurm/jwt_hs256.key bs=32 count=1
chmod 600 /etc/slurm/jwt_hs256.key

# 產生使用者權杖
scontrol token username=alice lifespan=3600
```

### 使用者限制

```bash
# 在 slurm.conf 中限制
AccountingStorageEnforce=associations,limits,qos
```

---

## 高可用性

### 備份控制器

```bash
# slurm.conf
SlurmctldHost=controller1(192.168.1.1)
SlurmctldHost=controller2(192.168.1.2)

# 狀態儲存（使用共用儲存）
StateSaveLocation=/shared/slurm/state
```

### 資料庫高可用

```bash
# 使用 MySQL 主從複寫
# 或使用 MariaDB Galera Cluster

# slurmdbd.conf 中設定備援
DbdBackupHost=backup-db-server
```

### 監控服務

```bash
# 使用 systemd 自動重啟
# /etc/systemd/system/slurmctld.service
[Service]
Restart=on-failure
RestartSec=10
```

---

## 相關文件

- [專案概覽](./project-overview.md) - Slurm 系統概述
- [架構文件](./architecture.md) - 詳細架構說明
- [資料模型](./data-models.md) - 資料結構說明
- [API 契約](./api-contracts.md) - REST API 文件
- [使用者指南](./user-guide.md) - 一般使用者文件
