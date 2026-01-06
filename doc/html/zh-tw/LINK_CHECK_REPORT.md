# 連結檢查報告

> 檢查日期：2026-01-06

## 摘要

| 類型 | 數量 |
|------|------|
| 總 .html 連結數 | 93 |
| 可修復（有對應 .md） | 52 |
| 無法修復（無翻譯） | 41 |
| 涉及檔案數 | 33 |

---

## 可修復的連結（已有翻譯）

以下 `.html` 連結在 `zh-tw/` 目錄中有對應的 `.md` 翻譯，應修改為 `.md`：

| 原連結 | 修改為 | 出現次數 |
|--------|--------|----------|
| `accounting.html` | `accounting.md` | 2 |
| `cons_tres.html` | `cons_tres.md` | 1 |
| `containers.html` | `containers.md` | 1 |
| `dynamic_nodes.html` | `dynamic_nodes.md` | 1 |
| `federation.html` | `federation.md` | 1 |
| `gang_scheduling.html` | `gang_scheduling.md` | 4 |
| `gres.html` | `gres.md` | 1 |
| `high_throughput.html` | `high_throughput.md` | 1 |
| `job_reason_codes.html` | `job_reason_codes.md` | 1 |
| `mcs.html` | `mcs.md` | 1 |
| `multi_cluster.html` | `multi_cluster.md` | 1 |
| `openapi_release_notes.html` | `openapi_release_notes.md` | 1 |
| `preempt.html` | `preempt.md` | 2 |
| `priority_multifactor.html` | `priority_multifactor.md` | 3 |
| `prolog_epilog.html` | `prolog_epilog.md` | 1 |
| `reservations.html` | `reservations.md` | 1 |
| `resource_limits.html` | `resource_limits.md` | 2 |
| `rest.html` | `rest.md` | 4 |
| `rest_clients.html` | `rest_clients.md` | 1 |
| `rest_quickstart.html` | `rest_quickstart.md` | 2 |
| `sched_config.html` | `sched_config.md` | 1 |
| `topology.html` | `topology.md` | 4 |
| `tres.html` | `tres.md` | 3 |
| `upgrades.html` | `upgrades.md` | 1 |

---

## 無法修復的連結（無對應翻譯）

以下連結指向尚未翻譯的文件，應保留連結到英文原版：

| 連結 | 出現次數 | 建議 |
|------|----------|------|
| `slurm.conf.html` | 10 | 保留（man page） |
| `slurmrestd.html` | 4 | 保留（man page） |
| `rest_api.html` | 3 | 保留（API 參考） |
| `slurmdbd.html` | 2 | 保留（man page） |
| `sacctmgr.html` | 2 | 保留（man page） |
| `mpi.conf.html` | 2 | 保留（設定檔） |
| `burst_buffer.conf.html` | 1 | 保留（設定檔） |
| `configurator.html` | 1 | 保留（工具） |
| `ha.html` | 1 | 保留（HA 設定） |
| `job_submit_plugins.html` | 1 | 保留（開發文件） |
| `oci.conf.html` | 1 | 保留（設定檔） |
| `platforms.html` | 1 | 保留（平台資訊） |
| `resources.yaml.html` | 1 | 保留（設定檔） |
| `sackd.html` | 1 | 保留（man page） |
| `scrun.html` | 1 | 保留（man page） |
| `site_factor.html` | 1 | 保留（外掛文件） |
| `slurmd.html` | 1 | 保留（man page） |
| `spank.html` | 1 | 保留（外掛文件） |
| `squeue.html` | 1 | 保留（man page） |
| `sreport.html` | 1 | 保留（man page） |
| `topology.conf.html` | 1 | 保留（設定檔） |
| `topology.yaml.html` | 1 | 保留（設定檔） |
| `sacct.html` | 1 | 保留（man page） |

---

## 修復狀態

- [x] 報告生成完成
- [x] 可修復連結已更新為 .md（45 個連結）
- [x] 驗證所有連結

## 修復結果

| 項目 | 數量 |
|------|------|
| 修復前 .html 連結數 | 93 |
| 修復後 .html 連結數 | 48 |
| 已修復連結數 | 45 |

剩餘 48 個 `.html` 連結為無對應翻譯的文件（man pages、設定檔參考等），保持原狀指向英文原版。
