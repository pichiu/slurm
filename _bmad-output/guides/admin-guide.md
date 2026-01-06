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

## 進階主題

以下為進階管理功能，詳細說明請參考官方文件。

### 升級指南

```bash
# 升級步驟
1. 備份設定和狀態檔案
2. 停止 slurmdbd
3. 停止 slurmctld
4. 停止所有 slurmd
5. 升級軟體套件
6. 更新設定檔（如有需要）
7. 啟動 slurmdbd
8. 啟動 slurmctld
9. 啟動所有 slurmd
```

📖 詳見：[升級指南](https://slurm.schedmd.com/upgrades.html)

### 容器支援

設定容器化作業執行：

```bash
# slurm.conf
# 啟用 OCI 容器執行時期
SwitchType=switch/none
MpiDefault=none

# 容器設定範例
# 使用 Singularity、Docker 或 Podman
```

📖 詳見：[容器指南](https://slurm.schedmd.com/containers.html)

### 無配置運行（Configless）

讓計算節點動態取得配置：

```bash
# slurm.conf (控制器端)
SlurmctldParameters=enable_configless

# 節點端啟動
slurmd --conf-server=controller:6817
```

📖 詳見：[無配置 Slurm 運行](https://slurm.schedmd.com/configless_slurm.html)

### Burst Buffer

高速暫存儲存管理：

```bash
# 作業腳本中使用
#BB create_persistent name=mybuffer capacity=100GB
#DW persistentdw name=mybuffer
```

📖 詳見：[Burst Buffer 指南](https://slurm.schedmd.com/burst_buffer.html)

### 聯邦排程

跨多個叢集的統一作業管理：

```bash
# 建立聯邦
sacctmgr add federation myfed clusters=cluster1,cluster2

# 作業會自動路由到最適合的叢集
```

📖 詳見：[聯邦排程指南](https://slurm.schedmd.com/federation.html)

### Prolog 和 Epilog

作業前後執行的腳本：

```bash
# slurm.conf
Prolog=/etc/slurm/prolog.sh
Epilog=/etc/slurm/epilog.sh
PrologSlurmctld=/etc/slurm/prolog_slurmctld.sh
EpilogSlurmctld=/etc/slurm/epilog_slurmctld.sh
```

📖 詳見：[Prolog 和 Epilog 指南](https://slurm.schedmd.com/prolog_epilog.html)

### 省電功能

自動關閉閒置節點：

```bash
# slurm.conf
SuspendProgram=/etc/slurm/suspend.sh
ResumeProgram=/etc/slurm/resume.sh
SuspendTime=600
ResumeTimeout=300
SuspendExcNodes=node[01-02]
```

📖 詳見：[省電指南](https://slurm.schedmd.com/power_save.html)

### 授權管理

管理軟體授權作為資源：

```bash
# slurm.conf
Licenses=matlab:50,ansys:10

# 作業請求
#SBATCH --licenses=matlab:2
```

📖 詳見：[授權管理](https://slurm.schedmd.com/licenses.html)

### pam_slurm_adopt

控制 SSH 連線到計算節點：

```bash
# /etc/pam.d/sshd
account required pam_slurm_adopt.so
```

📖 詳見：[pam_slurm_adopt 指南](https://slurm.schedmd.com/pam_slurm_adopt.html)

### 大型叢集管理

針對大型叢集的最佳化設定：

```bash
# slurm.conf
SchedulerParameters=bf_max_job_test=5000
MaxJobCount=100000
MaxArraySize=10001
TreeWidth=65535
```

📖 詳見：[大型叢集管理指南](https://slurm.schedmd.com/big_sys.html)

### 雲端部署

| 雲端平台 | 說明 |
|----------|------|
| Google Cloud Platform | 使用 slurm-gcp 專案 |
| AWS | AWS Parallel Computing Service |
| Microsoft Azure | CycleCloud 整合 |

📖 詳見：[雲端排程指南](https://slurm.schedmd.com/power_save.html)

---

## 官方文件參考

以下連結指向 Slurm 官方文件，提供更詳細的說明：

### 基礎管理
- [快速入門管理員指南](https://slurm.schedmd.com/quickstart_admin.html)
- [升級指南](https://slurm.schedmd.com/upgrades.html)
- [故障排除指南](https://slurm.schedmd.com/troubleshoot.html)
- [使用者權限](https://slurm.schedmd.com/user_permissions.html)

### 配置工具
- [配置工具（完整版）](https://slurm.schedmd.com/configurator.html)
- [配置工具（簡化版）](https://slurm.schedmd.com/configurator.easy.html)
- [無配置 Slurm 運行](https://slurm.schedmd.com/configless_slurm.html)

### 資源管理
- [記帳](https://slurm.schedmd.com/accounting.html)
- [進階資源預約指南](https://slurm.schedmd.com/reservations.html)
- [Cgroups 指南](https://slurm.schedmd.com/cgroups.html)
- [CPU 管理指南](https://slurm.schedmd.com/cpu_management.html)
- [動態節點](https://slurm.schedmd.com/dynamic_nodes.html)
- [授權管理](https://slurm.schedmd.com/licenses.html)
- [TRES（可追蹤資源）](https://slurm.schedmd.com/tres.html)

### 排程
- [排程配置指南](https://slurm.schedmd.com/sched_config.html)
- [消耗性資源指南](https://slurm.schedmd.com/cons_tres.html)
- [通用資源（GRES）排程](https://slurm.schedmd.com/gres.html)
- [高吞吐量計算指南](https://slurm.schedmd.com/high_throughput.html)
- [搶佔](https://slurm.schedmd.com/preempt.html)
- [服務品質（QoS）](https://slurm.schedmd.com/qos.html)
- [資源限制](https://slurm.schedmd.com/resource_limits.html)
- [拓樸](https://slurm.schedmd.com/topology.html)

### 優先級
- [多因素作業優先級](https://slurm.schedmd.com/priority_multifactor.html)
- [經典公平共用演算法](https://slurm.schedmd.com/classic_fair_share.html)
- [深度無關公平共用因素](https://slurm.schedmd.com/priority_multifactor3.html)
- [Fair Tree 公平共用演算法](https://slurm.schedmd.com/fair_tree.html)

### 安全與驗證
- [驗證外掛](https://slurm.schedmd.com/authentication.html)
- [JWT 驗證](https://slurm.schedmd.com/jwt.html)
- [多類別安全（MCS）指南](https://slurm.schedmd.com/mcs.html)
- [SELinux 上下文管理](https://slurm.schedmd.com/selinux.html)
- [pam_slurm_adopt 作業控制](https://slurm.schedmd.com/pam_slurm_adopt.html)

### 進階功能
- [Burst Buffer 指南](https://slurm.schedmd.com/burst_buffer.html)
- [容器](https://slurm.schedmd.com/containers.html)
- [聯邦排程指南](https://slurm.schedmd.com/federation.html)
- [Prolog 和 Epilog 指南](https://slurm.schedmd.com/prolog_epilog.html)
- [省電指南](https://slurm.schedmd.com/power_save.html)
- [大型叢集管理指南](https://slurm.schedmd.com/big_sys.html)
- [網路配置指南](https://slurm.schedmd.com/network.html)

### REST API
- [REST API 快速入門](https://slurm.schedmd.com/rest_quickstart.html)
- [REST API 詳細說明](https://slurm.schedmd.com/rest.html)
- [REST API 方法與模型](https://slurm.schedmd.com/rest_api.html)
- [REST API 客戶端指南](https://slurm.schedmd.com/rest_clients.html)

### 整合
- [Elasticsearch 指南](https://slurm.schedmd.com/elasticsearch.html)
- [Kubernetes 指南](https://slurm.schedmd.com/kubernetes.html)
- [NSS Slurm 名稱服務快取](https://slurm.schedmd.com/nss_slurm.html)
- [WCKey 管理](https://slurm.schedmd.com/wckey.html)

---

## 相關文件

- [專案概覽](./project-overview.md) - Slurm 系統概述
- [架構文件](./architecture.md) - 詳細架構說明
- [資料模型](./data-models.md) - 資料結構說明
- [API 契約](./api-contracts.md) - REST API 文件
- [使用者指南](./user-guide.md) - 一般使用者文件
- [開發者指南](./developer-guide.md) - 開發者文件
