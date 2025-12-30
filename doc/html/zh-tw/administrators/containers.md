# Slurm 容器指南 (Containers Guide)

## TL;DR

Slurm 原生支援為作業和步驟請求非特權 OCI 容器。設定步驟：設定核心和容器執行時、部署 oci.conf、重啟 slurmd、產生 OCI bundles。支援 runc、crun、nvidia-container-runtime、Singularity、Charliecloud、Enroot 等。也可透過 scrun 與 Rootless Docker 和 Podman 整合。

---

## 翻譯

### 概觀

容器正在 HPC 工作負載中被採用。容器依賴現有的核心功能，讓使用者更能控制應用程式在任何給定時間看到和可互動的內容。對於 HPC 工作負載，這些通常限於 mount namespace。Slurm 原生支援為作業和步驟請求非特權 OCI 容器。

#### 設定容器需要幾個步驟：

1. 設定核心和容器執行時
2. 部署適合的 oci.conf 檔案到計算節點可存取的位置
3. 在計算節點上重啟或重新設定 slurmd
4. 為所需的容器產生 OCI bundles 並放置在計算節點上
5. 驗證可以直接透過選擇的 OCI 執行時執行容器
6. 驗證可以透過 Slurm 請求容器

---

### 已知限制

- 所有容器必須在非特權（rootless）模式下執行
- 不支援自訂容器網路，所有容器應使用「host」網路
- Slurm 不會將 OCI 容器 bundle 傳輸到執行節點，bundle 必須已存在於執行節點上的請求路徑
- 容器受所使用的 OCI 執行時限制
- 執行節點上必須設定 oci.conf，否則 Slurm 會忽略請求的容器

---

### 先決條件

主機核心必須設定為允許使用者空間容器：

```bash
sudo sysctl -w kernel.unprivileged_userns_clone=1
sudo sysctl -w kernel.apparmor_restrict_unprivileged_unconfined=0
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

Docker 也提供工具來驗證核心設定：
```bash
dockerd-rootless-setuptool.sh check --force
```

---

### 必要軟體

- 完全功能的 OCI 執行時（需要能在 Slurm 外部先執行）
- 完全功能的 OCI bundle 產生工具

---

### 各種 OCI 執行時的設定範例

#### runc 使用 run（建議）

```
EnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeEnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeQuery="runc --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
RunTimeKill="runc --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
RunTimeDelete="runc --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
RunTimeRun="runc --rootless=true --root=/run/user/%U/ run %n.%u.%j.%s.%t -b %b"
```

#### crun 使用 run（建議）

```
EnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeEnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeQuery="crun --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
RunTimeKill="crun --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
RunTimeDelete="crun --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
RunTimeRun="crun --rootless=true --root=/run/user/%U/ run --bundle %b %n.%u.%j.%s.%t"
```

#### nvidia-container-runtime 使用 run（建議）

```
EnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeEnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeQuery="nvidia-container-runtime --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
RunTimeKill="nvidia-container-runtime --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
RunTimeDelete="nvidia-container-runtime --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
RunTimeRun="nvidia-container-runtime --rootless=true --root=/run/user/%U/ run %n.%u.%j.%s.%t -b %b"
```

#### Singularity v4.1.3 使用原生執行時

```
IgnoreFileConfigJson=true
EnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeEnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeRun="singularity exec --userns %r %@"
RunTimeKill="kill -s SIGTERM %p"
RunTimeDelete="kill -s SIGKILL %p"
```

#### Charliecloud (v0.30)

```
IgnoreFileConfigJson=true
CreateEnvFile=newline
EnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeEnvExclude="^(SLURM_CONF|SLURM_CONF_SERVER)="
RunTimeRun="env -i PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin/:/sbin/ USER=$(whoami) HOME=/home/$(whoami)/ ch-run -w --bind /etc/group:/etc/group --bind /etc/passwd:/etc/passwd --bind /etc/slurm:/etc/slurm --bind %m:/var/run/slurm/ --bind /var/run/munge/:/var/run/munge/ --set-env=%e --no-passwd %r -- %@"
RunTimeKill="kill -s SIGTERM %p"
RunTimeDelete="kill -s SIGKILL %p"
```

---

### 在 Slurm 外部測試 OCI 執行時

Slurm 直接在作業步驟中呼叫 OCI 執行時。如果失敗，作業也會失敗。

```bash
# 進入包含 OCI 容器 bundle 的目錄
cd $ABS_PATH_TO_BUNDLE

# 執行 OCI 容器執行時
$OCIRunTime $ARGS create test --bundle $PATH_TO_BUNDLE
$OCIRunTime $ARGS start test
$OCIRunTime $ARGS kill test
$OCIRunTime $ARGS delete test
```

如果這些命令成功，則 OCI 執行時已正確設定，可以在 Slurm 中測試。

---

### 請求容器作業或步驟

`salloc`、`srun` 和 `sbatch`（Slurm 21.08+）有 `--container` 參數，可用於請求容器執行時執行。

```bash
# 在容器內執行批次步驟
sbatch --container $ABS_PATH_TO_BUNDLE --wrap 'bash -c "cat /etc/*rel*"'

# 批次作業，步驟 0 在容器內
sbatch --wrap 'srun bash -c "--container $ABS_PATH_TO_BUNDLE cat /etc/*rel*"'

# 在容器內執行互動式步驟
salloc --container $ABS_PATH_TO_BUNDLE bash -c "cat /etc/*rel*"

# 互動式作業，步驟 0 在容器內
salloc srun --container $ABS_PATH_TO_BUNDLE bash -c "cat /etc/*rel*"

# 作業，步驟 0 在容器內
srun --container $ABS_PATH_TO_BUNDLE bash -c "cat /etc/*rel*"
```

**注意**：使用 `--container` 旗標執行的命令會在發送到容器*之前*透過 PATH 解析。如果容器有獨特的檔案結構，可能需要給出命令的完整路徑或指定 `--export=NONE` 讓容器定義要使用的 PATH：

```bash
srun --container $ABS_PATH_TO_BUNDLE --export=NONE bash -c "cat /etc/*rel*"
```

---

### 與 Rootless Docker 整合（Docker Engine v20.10+ & Slurm-23.02+）

Slurm 的 scrun 可以直接與 Rootless Docker 整合以作為作業執行容器。不需要特殊使用者權限。

#### 先決條件

1. slurm.conf 必須設定使用 Munge 認證
2. 必須設定 scrun.lua
3. 設定核心允許 ping
4. 設定 rootless dockerd 允許監聽特權通訊埠
5. oci.conf 必須存在於任何可能執行容器作業的節點上

#### 限制

- 不支援 JWT 認證
- Docker 容器建置目前不可用
- 所有容器必須使用「none」網路驅動程式
- 不支援 `docker exec`、`docker swarm`、`docker compose`、`docker pause` 命令

#### 設定程序

1. 安裝和設定 Rootless Docker
2. 設定環境：
   ```bash
   export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
   ```
3. 停止 rootless docker
4. 設定 Docker 呼叫 scrun：

   ```json
   {
       "experimental": true,
       "iptables": false,
       "bridge": "none",
       "no-new-privileges": true,
       "rootless": true,
       "selinux-enabled": false,
       "default-runtime": "slurm",
       "runtimes": {
           "slurm": {
               "path": "/usr/local/bin/scrun"
           }
       },
       "data-root": "/run/user/${USER_ID}/docker/",
       "exec-root": "/run/user/${USER_ID}/docker-exec/"
   }
   ```

5. 驗證 Docker 使用 scrun：
   ```bash
   export DOCKER_SECURITY="--security-opt label=disable --security-opt seccomp=unconfined --security-opt apparmor=unconfined --net=none"
   docker run $DOCKER_SECURITY hello-world
   docker run $DOCKER_SECURITY alpine /bin/printenv SLURM_JOB_ID
   ```

---

### 與 Podman 整合（Slurm-23.02+）

Slurm 的 scrun 可以直接與 Podman 整合以作為作業執行容器。

#### 設定 Podman 的 containers.conf

```ini
[containers]
apparmor_profile = "unconfined"
cgroupns = "host"
cgroups = "enabled"
default_sysctls = []
label = false
netns = "host"
no_hosts = true
pidns = "host"
utsns = "host"
userns = "host"
log_driver = "journald"

[engine]
cgroup_manager = "systemd"
runtime = "slurm"
remote = false

[engine.runtimes]
slurm = [
    "/usr/local/bin/scrun",
    "/usr/bin/scrun"
]
```

#### 驗證 Podman 使用 scrun

```bash
podman run hello-world
podman run alpine printenv SLURM_JOB_ID
podman run alpine hostname
```

---

### OCI 容器 Bundle

有多種方式產生 OCI 容器 bundle。以下是一些方法：

#### 使用 Docker 建立映像

```bash
mkdir -p ~/oci_images/alpine/rootfs
cd ~/oci_images/
docker pull alpine
docker create --name alpine alpine
docker export alpine | tar -C ~/oci_images/alpine/rootfs -xf -
docker rm alpine
```

#### 設定 bundle 供執行時執行

```bash
cd ~/oci_images/alpine
runc --rootless=true spec --rootless

# 測試執行映像
srun --container ~/oci_images/alpine/ uptime
```

#### 使用 umoci 和 skopeo

```bash
mkdir -p ~/oci_images/
cd ~/oci_images/
skopeo copy docker://alpine:latest oci:alpine:latest
umoci unpack --rootless --image alpine ~/oci_images/alpine
srun --container ~/oci_images/alpine uptime
```

---

### 透過外掛程式支援容器

Slurm 允許容器開發者建立 SPANK 外掛程式，可在作業執行的各個點呼叫以支援容器。使用這些外掛程式啟動容器的站台**不應**有 "oci.conf" 設定檔。

#### Shifter

Shifter 是來自 NERSC 的容器專案，提供具有完整排程器整合的 HPC 容器。

#### ENROOT 和 Pyxis

Enroot 是由 NVIDIA 贊助的使用者命名空間容器系統，支援：
- 透過 pyxis 進行 Slurm 整合
- 原生支援 Nvidia GPU
- 更快的 Docker 映像匯入

#### Sarus

Sarus 是由 ETH Zurich CSCS 贊助的特權容器系統，支援：
- 透過 OCI hook 進行 Slurm 映像同步
- 原生 OCI 映像支援
- NVIDIA GPU 支援

---

## 說明

### 容器執行時比較

| 執行時 | 特權需求 | GPU 支援 | 建議用途 |
|--------|----------|----------|----------|
| runc | 非特權 | 無 | 通用 OCI 容器 |
| crun | 非特權 | 無 | 輕量級替代 runc |
| nvidia-container-runtime | 非特權 | 原生 | NVIDIA GPU 工作負載 |
| Singularity | 視模式 | 支援 | HPC 專用 |
| Charliecloud | 非特權 | 支援 | 輕量級 HPC |
| Enroot | 非特權 | 原生 | NVIDIA 環境 |

### oci.conf 模式替換

| 模式 | 說明 |
|------|------|
| `%U` | 使用者 ID |
| `%u` | 使用者名稱 |
| `%n` | 節點名稱 |
| `%j` | 作業 ID |
| `%s` | 步驟 ID |
| `%t` | 任務 ID |
| `%b` | Bundle 路徑 |
| `%r` | 根檔案系統路徑 |
| `%@` | 命令和參數 |
| `%p` | 程序 ID |
| `%e` | 環境檔案路徑 |
| `%m` | Spool 目錄 |

---

## 實務範例

### 基本容器作業

```bash
# 準備 OCI bundle
mkdir -p ~/containers/alpine/rootfs
cd ~/containers/
docker pull alpine
docker create --name alpine alpine
docker export alpine | tar -C ~/containers/alpine/rootfs -xf -
docker rm alpine

# 產生設定
cd ~/containers/alpine
runc --rootless=true spec --rootless

# 在 Slurm 中執行
srun --container ~/containers/alpine/ hostname
sbatch --container ~/containers/alpine/ --wrap 'cat /etc/os-release'
```

### 多節點 MPI 容器作業

```bash
# 使用包含 OpenMPI 的容器
srun -N 4 --container ~/containers/openmpi/ mpirun hostname
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 使用特權模式執行容器 | 使用 --rootless=true |
| Bundle 不存在於執行節點 | 確保 bundle 在所有計算節點可存取 |
| 未設定 oci.conf | 在計算節點上部署 oci.conf |
| 使用 bridge 網路 | 使用 host 或 none 網路 |
| 命令路徑在容器中不存在 | 使用完整路徑或 --export=NONE |

### 除錯技巧

```bash
# 在 Slurm 外部測試執行時
runc --rootless=true --root=/run/user/$(id -u)/ run test -b /path/to/bundle

# 檢查容器日誌
journalctl -u slurmd | grep -i container

# 驗證 oci.conf 設定
cat /etc/slurm/oci.conf
```

---

## 快速參考

### 重要設定檔

| 檔案 | 位置 | 用途 |
|------|------|------|
| oci.conf | /etc/slurm/ | OCI 執行時設定 |
| scrun.lua | /etc/slurm/ | scrun 儲存設定 |
| containers.conf | ~/.config/containers/ | Podman 設定 |
| daemon.json | ~/.config/docker/ | Docker 設定 |

### 容器命令選項

| 選項 | 說明 |
|------|------|
| `--container` | 指定 OCI bundle 路徑 |
| `--export=NONE` | 不繼承環境變數 |

### 核心設定

```bash
# 啟用非特權容器
sysctl -w kernel.unprivileged_userns_clone=1
sysctl -w kernel.apparmor_restrict_unprivileged_unconfined=0
sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

### 相關文件

- [oci.conf](oci.conf.html) - OCI 設定檔參考
- [scrun](scrun.html) - Slurm 容器執行時
- [GRES](gres.md) - GPU 設定（容器中使用 GPU）
