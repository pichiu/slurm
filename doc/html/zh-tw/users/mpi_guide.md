# Slurm MPI 使用者指南

## TL;DR
- Slurm 支援透過 PMI-1、PMI-2 或 PMIx API 啟動 MPI 作業
- 推薦使用 `srun` 直接啟動 MPI 作業以獲得完整的會計和親和性功能
- 使用 `--mpi=` 選項或 `MpiDefault` 配置選擇 PMI 類型
- 支援 Open MPI、Intel MPI、MPICH、MVAPICH2 和 HPE Cray PMI
- 確保 MPI 程式庫與 Slurm 的 PMI 外掛程式匹配

---

## Translation

### 概述

MPI 的使用取決於所使用的 MPI 類型。這些各種 MPI 實作使用三種根本不同的操作模式：

1. Slurm 直接啟動任務並透過 PMI-1、PMI-2 或 PMIx API 執行通訊初始化。（大多數現代 MPI 實作都支援。）

2. Slurm 為作業建立資源分配，然後 mpirun 使用 Slurm 的基礎設施 (srun) 啟動任務。

3. Slurm 為作業建立資源分配，然後 mpirun 使用 Slurm 以外的某些機制啟動任務，例如 SSH 或 RSH。這些任務在 Slurm 的監控或控制之外啟動，需要從批次節點存取節點（例如 SSH）。Slurm 的 epilog 應配置為在作業的分配被放棄時清除這些任務。強烈建議使用 pam_slurm_adopt。

**注意**：在第 3 種情況下，Slurm 不是直接啟動使用者應用程式，這可能會阻止將任務繫結到 CPU 和/或會計的預期行為，不是推薦的方式。

兩個 Slurm 參數控制將支援哪種 PMI（程序管理介面）實作。正確的配置對於 Slurm 為 MPI 作業建立適當的環境至關重要，例如設定適當的環境變數。slurm.conf 中的 *MpiDefault* 配置參數建立系統要使用的預設 PMI。*srun* 選項 *--mpi=*（或等效的環境變數 *SLURM_MPI_TYPE*）可用於在為個別作業使用不同的 PMI 實作時指定。

[mpi.conf](mpi.conf.html) 檔案中有一些參數允許您修改 PMI 外掛程式的行為。

**注意**：使用沒有適當 Slurm 外掛程式的 MPI 實作可能會導致應用程式失敗。如果系統上使用多個 MPI 實作，則可能需要某些使用者明確指定合適的 Slurm MPI 外掛程式。

**注意**：如果使用 RPM 安裝 Slurm，*slurm-libpmi* 套件如果已安裝 *pmix-libpmi* 套件會發生衝突。如果您的站點策略允許從原始碼安裝，這將允許您將這些套件安裝到不同的位置，以便您可以選擇使用哪些程式庫。

**注意**：如果您使用 hwloc 建置任何 MPI 堆疊組件，請注意 hwloc 版本 2.5.0 到 2.7.0（含）有一個錯誤，會在 environ 陣列中推入一個不可觸及的值，導致存取時發生段錯誤。建議使用 hwloc 版本 2.7.1 或更高版本進行建置。

### PMIx

#### 建置 PMIx

在建置 PMIx 之前，建議閱讀這些 [How-To 指南](https://openpmix.github.io/support/how-to/)。它們提供了關於[建置依賴項和安裝步驟](https://openpmix.github.io/code/getting-the-reference-implementation)的一些詳細資訊，以及關於 [Slurm 支援](https://openpmix.github.io/support/how-to/slurm-support)的相關說明。

PMIx 可以從官方 [PMIx GitHub](https://github.com/openpmix/openpmix) 儲存庫獲得，可以透過克隆儲存庫或下載打包的版本。

Slurm 對 PMIx 的支援首次包含在基於 PMIx v1.2 版本的 Slurm 16.05 中。此後已更新為支援 PMIx 系列的最高版本 5.x，如下表所示：
- Slurm 20.11+ 支援 PMIx v1.2+、v2.x 和 v3.x。
- Slurm 22.05+ 支援 PMIx v2.x、v3.x、v4.x 和 v5.x。

如果執行 PMIx v1，建議至少執行 1.2.5，因為較舊的版本可能與 pmi 和 pmi2 API 的支援有一些相容性問題。

另請注意，Intel MPI 不正式支援 PMIx。由於 PMIx 提供與 PMI-2 的一些相容性，它可能會工作，但不能保證。

#### 使用 PMIx 支援建置 Slurm

在配置時，除非設定 **--with-pmix**，否則 Slurm 不會使用 PMIx 建置。然後它將預設在以下位置尋找 PMIx 安裝：

```
/usr
/usr/local
```

如果 PMIx 沒有安裝在上述任何位置，可以請求 Slurm 配置腳本指向非預設位置。以下是假設安裝目錄為 /home/user/pmix/v4.1.2/ 的範例：

```bash
user@testbox:~/slurm/22.05/build$ ../src/configure \
> --prefix=/home/user/slurm/22.05/inst \
> --with-pmix=/home/user/pmix/4.1.2
```

或使用基於 RPM 的建置的類似方式：

```bash
user@testbox:~/slurm_rpm$ rpmbuild \
> --define '_prefix /home/user/slurm/22.05/inst' \
> --define '_slurm_sysconfdir /home/user/slurm/22.05/inst/etc' \
> --define '_with_pmix --with-pmix=/home/user/pmix/4.1.2' \
> -ta slurm-22.05.2.1.tar.bz2
```

**注意**：也可以使用 ':' 分隔符針對多個 PMIx 版本建置。例如，要針對 3.2 和 4.1 建置：

```bash
--with-pmix=/path/to/pmix/3.2.3:/path/to/pmix/4.1.2
```

然後，在提交作業時，可以使用 --mpi=list 中可用的任何版本選擇所需的版本。pmix 的預設值將是程式庫的最高版本：

```bash
$ srun --mpi=list
MPI plugin types are...
	cray_shasta
	none
	pmi2
	pmix
specific pmix plugin versions available: pmix_v3,pmix_v4
```

如果還需要 PMI-1 或 PMI-2 版本的支援，也可以從 contribs 目錄安裝：

```bash
user@testbox:~/slurm/22.05/build/$ cd contribs/pmi1
user@testbox:~/slurm/22.05/build/contribs/pmi1$ make -j install

user@testbox:~/slurm/22.05/build/$ cd contribs/pmi2
user@testbox:~/slurm/22.05/build/contribs/pmi2$ make -j install
```

**注意**：由於 Slurm 和低於 4.x 的 PMIx 都提供 libpmi[2].so 程式庫，我們建議您將兩個軟體安裝在不同的位置。否則，這些相同的程式庫可能最終會安裝在 /usr/lib64 等標準位置下，套件管理器會報告衝突並出錯。

**注意**：任何針對 PMIx 編譯的應用程式都應使用與 Slurm 使用的相同 PMIx 或至少具有相同安全域的 PMIx，否則可能會有認證問題。

#### 測試 Slurm 和 PMIx

可以直接測試 Slurm 和 PMIx，而不需要安裝 MPI 實作：

```bash
$ srun --mpi=list
MPI plugin types are...
	cray_shasta
	none
	pmi2
	pmix
specific pmix plugin versions available: pmix_v3,pmix_v4

$ srun --mpi=pmix_v4 -n2 -N2 \
> /home/user/git/pmix/test/pmix_client -n 2 --job-fence -c
==141756== OK
==141774== OK
```

### Open MPI

當前版本的 Slurm 和 Open MPI 支援使用 srun 指令啟動任務。

如果 OpenMPI 配置為 *--with-pmi=* 指向 Slurm 的 PMI-1 libpmi.so 或 PMI-2 libpmi2.so 程式庫，則可以使用 srun 指令直接啟動 OMPI 作業。這是首選的操作模式，因為 Slurm 完成的會計功能和親和性將變得可用。如果啟用了 pmi2 支援，則必須在 srun 命令列上指定選項「--mpi=pmi2」。或者在 slurm.conf 中配置「MpiDefault=pmi」或「MpiDefault=pmi2」。

從 Open MPI 版本 3.1 開始，原生支援 PMIx。要使用 PMIx 啟動 Open MPI 應用程式，必須在 srun 命令列上指定「--mpi=pmix」選項，或在 slurm.conf 中配置「MpiDefault=pmix」。

也可以使用外部 PMIx 安裝建置 OpenMPI。請參閱 OpenMPI 文件以獲取詳細程序，但基本上包括在配置 OpenMPI 時指定 **--with-pmix=PATH**。

有一組參數可用於控制 Slurm PMIx 外掛程式的行為，請閱讀 [mpi.conf](mpi.conf.html) 以獲取更多資訊。

**注意**：OpenMPI 有一個限制，不支援從 Slurm 分配中呼叫 *MPI_Comm_spawn()*。如果您需要使用 *MPI_Comm_spawn()* 函數，您需要使用另一個 MPI 實作與 PMI-2 結合，因為 PMIx 也不支援它。

**注意**：某些核心和系統配置導致鎖定記憶體對於適當的 OpenMPI 功能來說太小，導致應用程式因段錯誤而失敗。可以透過配置 slurmd 守護程序以更大的限制執行來解決此問題。例如，在您的 slurmd.service 檔案中添加「LimitMEMLOCK=infinity」。

### Intel MPI

Intel® MPI Library for Linux OS 支援以下在 Slurm 作業管理器控制下啟動 MPI 作業的方法：

1. 透過 Hydra PM 的 mpirun 指令
2. srun 指令（Slurm，推薦）

#### 透過 Hydra 程序管理器的 mpirun 指令

Intel® MPI Library 的 mpirun 指令預設透過 Hydra 程序管理器支援 Slurm。在分配中啟動時，mpirun 指令將自動讀取 Slurm 設定的環境變數，如節點、CPU、任務等，以便在每個節點上啟動所需的 hydra 守護程序。這些守護程序將使用 srun 啟動，隨後將啟動使用者應用程式。

由於 Intel® MPI 僅支援 PMI-1 和 PMI-2（不支援 PMIx），強烈建議配置此 MPI 實作以使用 Slurm 的 PMI-2，它提供比 PMI-1 更好的可擴展性。不建議使用 PMI-1，應該很快就會被棄用。

以下是如何使用從 contribs 安裝的 Slurm PMI-2 程式庫在 10 個節點的獨佔分配中啟動使用者應用程式的範例：

```bash
$ salloc -N10 --exclusive
$ export I_MPI_PMI_LIBRARY=/path/to/slurm/lib/libpmi2.so
$ mpirun -np <num_procs> user_app.bin
```

#### srun 指令（Slurm，推薦）

這種方法也受 Intel® MPI Library 支援。這種方法與 Slurm 整合得最好，支援程序追蹤、會計、任務親和性、暫停/恢復和其他功能。

```bash
$ salloc -N10 --exclusive
$ export I_MPI_PMI_LIBRARY=/path/to/slurm/lib/libpmi2.so
$ srun user_app.bin
```

**注意**：我們手動指向 Slurm 的 PMI-1 或 PMI-2 程式庫是出於授權原因。IMPI 不直接連結到任何外部 PMI 實作，因此與其他堆疊（OMPI、MPICH、MVAPICH...）不同，Intel 不是針對 Slurm 程式庫建置的。指向此程式庫將導致 Intel dlopen 並使用此 PMI 程式庫。

**注意**：Intel 對 PMIx 程式庫沒有官方支援。由於 IMPI 基於 MPICH，使用 PMIx 與 Intel 可能會工作，因為 PMIx 維護與 pmi2（MPICH 中使用的程式庫）的相容性，但不能保證在所有情況下都能執行，PMIx 可能會在未來版本中破壞此相容性。

### MPICH

MPICH 以前稱為 MPICH2。MPICH 作業可以使用 **srun** 或 **mpiexec** 啟動。MPICH 實作支援 PMI-1、PMI-2 和 PMIx（從 MPICH v4 開始）。

#### MPICH 使用 srun 並連結 Slurm 的 PMI-1 或 PMI-2 程式庫

MPICH 可以專門為與 Slurm 及其 PMI-1 或 PMI-2 程式庫一起使用而建置：

```bash
# 對於 PMI-2：
user@testbox:~/mpich-4.0.2/build$ LD_LIBRARY_PATH=~/slurm/22.05/inst/lib/ \
> ../configure --prefix=/home/user/bin/mpich/ --with-pmilib=slurm \
> --with-pmi=pmi2 --with-slurm=/home/user/slurm/22.05/inst

# 或對於 PMI-1：
user@testbox:~/mpich-4.0.2/build$ LD_LIBRARY_PATH=~/slurm/22.05/inst/lib/ \
> ../configure --prefix=/home/user/bin/mpich/ --with-pmilib=slurm \
> --with-slurm=/home/user/slurm/22.05/inst
```

由於 PMI-1 已經過時且擴展性不佳，我們不建議您連結它。最好使用 PMI-2：

```bash
$ mpicc -o hello_world hello_world.c
$ srun --mpi=pmi2 ./hello_world
```

#### MPICH 使用 PMIx 並與 Slurm 整合

您也可以使用外部 PMIx 程式庫建置 MPICH，該程式庫應與建置 Slurm 時使用的相同：

```bash
$ LD_LIBRARY_PATH=~/slurm/22.05/inst/lib/ ../configure \
> --prefix=/home/user/bin/mpich/ \
> --with-pmix=/home/user/bin/pmix_4.1.2/ \
> --with-pmi=pmix \
> --with-slurm=/home/user/slurm/22.05/inst
```

編譯並執行程序：

```bash
$ mpicc -o hello_world hello_world.c
$ srun --mpi=pmix ./hello_world
```

### MVAPICH2

MVAPICH2 支援 Slurm。要啟用它，您需要使用類似以下的指令建置 MVAPICH2：

```bash
$ ./configure --prefix=/home/user/bin/mvapich2 \
> --with-slurm=/home/user/slurm/22.05/inst/
```

當 MVAPICH2 使用 Slurm 支援建置時，它將檢測到它在 Slurm 分配中，並將使用「srun」指令生成其 hydra 守護程序。它不連結到 Slurm API，這意味著在 Slurm 升級期間不需要重新編譯 MVAPICH2。

#### MVAPICH2 使用 srun 並連結 Slurm 的 PMI-1 或 PMI-2 程式庫

```bash
# 對於 PMI-2：
./configure --prefix=/home/user/bin/mvapich2 \
> --with-slurm=/home/user/slurm/22.05/inst/ \
> --with-pm=slurm --with-pmi=pmi2

# 對於 PMI-1：
./configure --prefix=/home/user/bin/mvapich2 \
> --with-slurm=/home/user/slurm/22.05/inst/ \
> --with-pm=slurm --with-pmi=pmi1
```

編譯並在 Slurm 中執行使用者應用程式：

```bash
$ mpicc -o hello_world hello_world.c
$ srun --mpi=pmi2 ./hello_world
```

#### MVAPICH2 使用 PMIx 並與 Slurm 整合

```bash
./configure --prefix=/home/user/bin/mvapich2 \
> --with-slurm=/home/user/slurm/22.05/inst/ \
> --with-pm=slurm \
> --with-pmix=/home/user/bin/pmix_4.1.2/ \
> --with-pmi=pmix
```

執行作業：

```bash
$ mpicc -o hello_world hello_world.c
$ srun --mpi=pmix ./hello_world
```

### HPE Cray PMI 支援

Slurm 預設附帶一個 Cray PMI 供應商特定外掛程式，提供與 HPE Cray 程式設計環境的 PMI 的相容性。它旨在用於在 HPE Cray 機器上使用此環境建置的應用程式。

該外掛程式名為 *cray_shasta*（Shasta 是此外掛程式支援的第一個 Cray 架構），並在所有 Slurm 安裝中預設建置：

```bash
$ srun --mpi=list
MPI plugin types are...
	cray_shasta
	none
```

Cray PMI 外掛程式將使用一些保留的連接埠進行通訊。這些連接埠可以使用 **srun** 的命令列選項 *--resv-ports* 進行配置，或透過在 slurm.conf 中設定 *MpiParams=ports*=[*port_range*] 進行配置。

此外掛程式不支援 MPMD/異質作業，並且需要 *libpals >= 0.2.8*。

---

## Explanation

### MPI 啟動模式比較

| 模式 | 方法 | 優點 | 缺點 |
|------|------|------|------|
| srun 直接啟動 | `srun --mpi=<type> ./app` | 完整會計、親和性、資源追蹤 | 需要正確配置 PMI |
| mpirun + Slurm | `mpirun -np N ./app` | 熟悉的介面 | 需要 Hydra 與 Slurm 整合 |
| mpirun + SSH | `mpirun -np N ./app` | 不需要 Slurm 整合 | 無會計、無資源追蹤 |

### PMI 類型比較

| PMI 類型 | 說明 | 建議 |
|----------|------|------|
| PMI-1 | 原始 PMI 標準 | 不建議，擴展性差 |
| PMI-2 | 改進的 PMI 標準 | 建議用於 Intel MPI |
| PMIx | 新一代 PMI 標準 | 建議用於現代 MPI 實作 |

---

## Practical Example

### 場景：在 Slurm 叢集上執行 MPI 作業

```bash
# 1. 查看可用的 MPI 外掛程式
$ srun --mpi=list
MPI plugin types are...
	none
	pmi2
	pmix
specific pmix plugin versions available: pmix_v4

# 2. 編譯 MPI 程式
$ mpicc -o hello_mpi hello_mpi.c

# 3. 使用 srun 直接啟動（推薦）
$ srun -N4 -n16 --mpi=pmix ./hello_mpi
# -N4       使用 4 個節點
# -n16      執行 16 個任務
# --mpi=pmix 使用 PMIx

# 4. 或使用批次作業
$ cat mpi_job.sh
#!/bin/bash
#SBATCH -N 4
#SBATCH -n 16
#SBATCH --time=1:00:00

module load openmpi
srun --mpi=pmix ./hello_mpi

$ sbatch mpi_job.sh
```

### 使用 Intel MPI

```bash
# 設定 PMI 程式庫路徑
$ export I_MPI_PMI_LIBRARY=/path/to/slurm/lib/libpmi2.so

# 方法 1：使用 srun（推薦）
$ salloc -N4 --exclusive
$ srun ./my_intel_mpi_app

# 方法 2：使用 mpirun
$ salloc -N4 --exclusive
$ mpirun -np 16 ./my_intel_mpi_app
```

### 測試 PMIx 配置

```bash
# 驗證 PMIx 外掛程式可用
$ srun --mpi=list | grep pmix
	pmix
specific pmix plugin versions available: pmix_v4

# 使用 PMIx 測試程式（不需要 MPI）
$ srun --mpi=pmix_v4 -n2 -N2 /path/to/pmix/test/pmix_client -n 2 --job-fence -c
==12345== OK
==12346== OK
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **MPI 外掛程式不匹配**
   ```bash
   # 錯誤：使用錯誤的 MPI 類型
   $ srun --mpi=pmi2 ./openmpi_app  # 但 OpenMPI 編譯時使用 PMIx
   # 可能導致應用程式失敗

   # 正確：匹配 MPI 編譯時的 PMI 類型
   $ srun --mpi=pmix ./openmpi_app
   ```

2. **忘記設定 PMI 程式庫路徑（Intel MPI）**
   ```bash
   # 錯誤：直接執行
   $ srun ./intel_mpi_app
   # 可能使用錯誤的 PMI 程式庫

   # 正確：設定程式庫路徑
   $ export I_MPI_PMI_LIBRARY=/path/to/slurm/lib/libpmi2.so
   $ srun ./intel_mpi_app
   ```

3. **版本不匹配**
   ```bash
   # 問題：PMIx 版本不匹配導致認證問題
   # Slurm 使用 PMIx 4.x，但應用程式使用 PMIx 3.x

   # 解決：確保使用相同的 PMIx 版本
   # 或設定 PMIX_MCA_psec=native
   ```

4. **鎖定記憶體限制太低**
   ```bash
   # 問題：OpenMPI 因段錯誤失敗
   # 原因：鎖定記憶體限制太小

   # 解決：在 slurmd.service 添加
   LimitMEMLOCK=infinity
   ```

### 實用建議

- **優先使用 srun**：提供完整的會計、親和性和資源追蹤功能
- **使用 PMIx**：現代 MPI 實作的首選，提供更好的可擴展性
- **配置 MpiDefault**：在 slurm.conf 設定預設 MPI 類型，減少命令列選項
- **驗證配置**：使用 `srun --mpi=list` 檢查可用的 MPI 外掛程式
- **相同的 PMIx 版本**：確保 MPI 程式庫和 Slurm 使用相同的 PMIx 版本

---

## Quick Reference

### srun MPI 選項

| 選項 | 說明 |
|------|------|
| `--mpi=list` | 列出可用的 MPI 外掛程式 |
| `--mpi=pmix` | 使用 PMIx |
| `--mpi=pmi2` | 使用 PMI-2 |
| `--mpi=none` | 不使用 MPI 外掛程式 |

### 常用環境變數

| 變數 | 說明 |
|------|------|
| `SLURM_MPI_TYPE` | 設定 MPI 類型（等同 --mpi=）|
| `I_MPI_PMI_LIBRARY` | Intel MPI 的 PMI 程式庫路徑 |
| `HYDRA_BOOTSTRAP` | Hydra 啟動機制 |
| `PMIX_MCA_psec` | PMIx 安全方法 |

### slurm.conf MPI 相關配置

| 參數 | 說明 |
|------|------|
| `MpiDefault=` | 預設 MPI 類型 |
| `MpiParams=` | MPI 參數設定 |

### MPI 實作支援的 PMI 類型

| MPI 實作 | PMI-1 | PMI-2 | PMIx |
|----------|-------|-------|------|
| Open MPI | ✓ | ✓ | ✓ (3.1+) |
| Intel MPI | ✓ | ✓ | (非官方) |
| MPICH | ✓ | ✓ | ✓ (v4+) |
| MVAPICH2 | ✓ | ✓ | ✓ |
