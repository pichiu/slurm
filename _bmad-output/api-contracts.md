# Slurm REST API Contracts

> Generated: 2025-12-17 | Source: slurmrestd OpenAPI plugins

## Overview

Slurm provides a comprehensive REST API through `slurmrestd` daemon, offering HTTP/HTTPS access to cluster management functionality. The API is organized into two main plugins:

1. **openapi/slurmctld** - Controller operations (jobs, nodes, partitions)
2. **openapi/slurmdbd** - Database operations (accounting, users, QoS)

**Base Port**: 6820 (configurable)
**Authentication**: JWT tokens or local Unix socket

---

## slurmctld API Endpoints (`/slurm/{data_parser}/...`)

### Jobs

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurm/{data_parser}/jobs/` | Get list of jobs |
| DELETE | `/slurm/{data_parser}/jobs/` | Send signal to list of jobs |
| GET | `/slurm/{data_parser}/jobs/state/` | Get list of job states |
| GET | `/slurm/{data_parser}/job/{job_id}` | Get job info |
| POST | `/slurm/{data_parser}/job/{job_id}` | Update job |
| DELETE | `/slurm/{data_parser}/job/{job_id}` | Cancel or signal job |
| POST | `/slurm/{data_parser}/job/submit` | Submit new job |
| POST | `/slurm/{data_parser}/job/allocate` | Submit new job allocation |

### Nodes

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurm/{data_parser}/nodes/` | Get node(s) info |
| POST | `/slurm/{data_parser}/nodes/` | Batch update node(s) |
| GET | `/slurm/{data_parser}/node/{node_name}` | Get node info |
| POST | `/slurm/{data_parser}/node/{node_name}` | Update node properties |
| DELETE | `/slurm/{data_parser}/node/{node_name}` | Delete node |
| POST | `/slurm/{data_parser}/new/node/` | Create node |

### Partitions

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurm/{data_parser}/partitions/` | Get all partition info |
| GET | `/slurm/{data_parser}/partition/{partition_name}` | Get partition info |

### Reservations

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurm/{data_parser}/reservations/` | Get all reservation info |
| POST | `/slurm/{data_parser}/reservations/` | Create or update reservations |
| GET | `/slurm/{data_parser}/reservation/{reservation_name}` | Get reservation info |
| POST | `/slurm/{data_parser}/reservation` | Create or update a reservation |
| DELETE | `/slurm/{data_parser}/reservation/{reservation_name}` | Delete a reservation |

### System

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurm/{data_parser}/ping/` | Ping test |
| GET | `/slurm/{data_parser}/diag/` | Get diagnostics |
| GET | `/slurm/{data_parser}/reconfigure/` | Request slurmctld reconfigure |
| GET | `/slurm/{data_parser}/licenses/` | Get all Slurm tracked license info |
| GET | `/slurm/{data_parser}/shares` | Get fairshare info |
| GET | `/slurm/{data_parser}/resources/{job_id}` | Get resource layout info |

---

## slurmdbd API Endpoints (`/slurmdb/{data_parser}/...`)

### Jobs (Accounting)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/jobs/` | Get job list |
| POST | `/slurmdb/{data_parser}/jobs/` | Update jobs |
| GET | `/slurmdb/{data_parser}/job/{job_id}` | Get job info |
| POST | `/slurmdb/{data_parser}/job/{job_id}` | Update job |

### Users

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/users/` | Get user list |
| POST | `/slurmdb/{data_parser}/users/` | Update users |
| GET | `/slurmdb/{data_parser}/user/{name}` | Get user info |
| DELETE | `/slurmdb/{data_parser}/user/{name}` | Delete user |
| POST | `/slurmdb/{data_parser}/users_association/` | Add users with conditional association |

### Accounts

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/accounts/` | Get account list |
| POST | `/slurmdb/{data_parser}/accounts/` | Add/update list of accounts |
| GET | `/slurmdb/{data_parser}/account/{account_name}` | Get account info |
| DELETE | `/slurmdb/{data_parser}/account/{account_name}` | Delete account |
| POST | `/slurmdb/{data_parser}/accounts_association/` | Add accounts with conditional association |

### Associations

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/associations/` | Get association list |
| POST | `/slurmdb/{data_parser}/associations/` | Set associations info |
| DELETE | `/slurmdb/{data_parser}/associations/` | Delete associations |
| GET | `/slurmdb/{data_parser}/association/` | Get association info |
| DELETE | `/slurmdb/{data_parser}/association/` | Delete association |

### QoS (Quality of Service)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/qos/` | Get QOS list |
| POST | `/slurmdb/{data_parser}/qos/` | Add or update QOSs |
| GET | `/slurmdb/{data_parser}/qos/{qos}` | Get QOS info |
| DELETE | `/slurmdb/{data_parser}/qos/{qos}` | Delete QOS |

### Clusters

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/clusters/` | Get cluster list |
| POST | `/slurmdb/{data_parser}/clusters/` | Modify clusters |
| GET | `/slurmdb/{data_parser}/cluster/{cluster_name}` | Get cluster info |
| DELETE | `/slurmdb/{data_parser}/cluster/{cluster_name}` | Delete cluster |

### TRES (Trackable Resources)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/tres/` | Get TRES info |
| POST | `/slurmdb/{data_parser}/tres/` | Add TRES |

### WCKeys

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/wckeys/` | Get wckey list |
| POST | `/slurmdb/{data_parser}/wckeys/` | Add or update wckeys |
| GET | `/slurmdb/{data_parser}/wckey/{id}` | Get wckey info |
| DELETE | `/slurmdb/{data_parser}/wckey/{id}` | Delete wckey |

### Instances

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/instances/` | Get instance list |
| GET | `/slurmdb/{data_parser}/instance/` | Get instance info |

### System

| Method | Path | Summary |
|--------|------|---------|
| GET | `/slurmdb/{data_parser}/ping/` | Ping test |
| GET | `/slurmdb/{data_parser}/diag/` | Get slurmdb diagnostics |
| GET | `/slurmdb/{data_parser}/config` | Dump all configuration |
| POST | `/slurmdb/{data_parser}/config` | Load all configuration |

---

## Data Parser Versions

The `{data_parser}` path parameter specifies the API version:
- `v0.0.42` - Slurm 24.11
- `v0.0.43` - Slurm 25.05
- `v0.0.44` - Slurm 25.11
- `v0.0.45` - Slurm 26.05 (current)

---

## Authentication

### JWT Token Authentication
```bash
curl -H "X-SLURM-USER-TOKEN: $JWT_TOKEN" \
     http://localhost:6820/slurm/v0.0.45/jobs/
```

### Local Unix Socket
```bash
curl --unix-socket /var/run/slurmrestd.socket \
     http://localhost/slurm/v0.0.45/jobs/
```

---

## Response Format

All responses follow OpenAPI format with:
- `meta` - Response metadata (plugin, slurm version)
- `errors` - Array of error objects
- `warnings` - Array of warning objects
- Data-specific fields based on endpoint

Example:
```json
{
  "meta": {
    "plugin": {"type": "openapi/slurmctld", "name": "Slurm OpenAPI slurmctld"},
    "slurm": {"version": {"major": 26, "minor": 5, "micro": 0}, "release": "26.05.0"}
  },
  "errors": [],
  "warnings": [],
  "jobs": [...]
}
```

---

## Source Files

| Plugin | Location |
|--------|----------|
| slurmctld OpenAPI | `src/slurmrestd/plugins/openapi/slurmctld/api.c` |
| slurmdbd OpenAPI | `src/slurmrestd/plugins/openapi/slurmdbd/api.c` |
| REST Auth | `src/slurmrestd/rest_auth.c` |
| HTTP Handler | `src/slurmrestd/http.c` |
