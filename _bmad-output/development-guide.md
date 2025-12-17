# Slurm Development Guide

> Generated: 2025-12-17

## Prerequisites

### Required Dependencies

```bash
# Build tools
autoconf >= 2.59
automake
gcc (or clang)
make
libtool
pkg-config

# Core libraries
munge-devel       # Authentication (recommended)
mysql-devel       # OR mariadb-devel for accounting
readline-devel    # Interactive CLI
python3           # Build scripts
perl-devel        # Perl API

# Optional dependencies
hwloc-devel       # Hardware topology
numactl-devel     # NUMA support
lua-devel >= 5.1  # Lua scripting
pam-devel         # PAM support
check >= 0.9.8    # Unit testing
pmix-devel        # PMIx MPI support
ucx-devel         # UCX networking
libcurl-devel     # HTTP support
libjwt-devel      # JWT authentication
systemd-devel     # Systemd integration
```

### Platform Support

- **Primary**: Linux (tested extensively)
- **Architectures**: x86_64, ARM64, POWER
- **Distributions**: RHEL/CentOS, Ubuntu/Debian, SUSE, Fedora

---

## Build Instructions

### Standard Build

```bash
# Clone repository
git clone https://github.com/SchedMD/slurm.git
cd slurm

# Configure (detect features automatically)
./configure --prefix=/usr/local

# Build
make -j$(nproc)

# Install (as root)
sudo make install
```

### Development Build

```bash
# Enable debug symbols and disable optimization
./configure --prefix=/usr/local \
            --enable-debug \
            --enable-developer \
            CFLAGS="-g -O0"

make -j$(nproc)
```

### Configure Options

| Option | Description |
|--------|-------------|
| `--enable-debug` | Enable debug output and symbols |
| `--enable-developer` | Enable developer mode checks |
| `--enable-multiple-slurmd` | Allow multiple slurmd per node |
| `--with-munge` | Enable MUNGE authentication |
| `--with-jwt` | Enable JWT authentication |
| `--with-pmix=PATH` | PMIx installation path |
| `--with-ucx=PATH` | UCX installation path |
| `--with-hwloc=PATH` | hwloc installation path |
| `--with-lua=PATH` | Lua installation path |
| `--with-slurmrestd` | Build REST API daemon |
| `--with-yaml` | Enable YAML serializer |
| `--without-pam` | Disable PAM support |

### RPM Build

```bash
# Create tarball
make dist

# Build RPMs
rpmbuild -ta slurm-*.tar.bz2

# With options
rpmbuild -ta slurm-*.tar.bz2 \
         --with slurmrestd \
         --with jwt \
         --with pmix
```

---

## Project Structure

```
slurm/
├── configure.ac      # Autoconf input
├── Makefile.am       # Automake template
├── src/              # Source code
│   ├── api/          # Client API
│   ├── common/       # Shared libraries
│   ├── slurmctld/    # Controller daemon
│   ├── slurmd/       # Node daemon
│   ├── slurmdbd/     # Database daemon
│   ├── slurmrestd/   # REST daemon
│   ├── s*/           # CLI tools
│   └── plugins/      # Plugin system
├── slurm/            # Public headers
├── doc/              # Documentation
├── testsuite/        # Tests
└── contribs/         # Contributed modules
```

---

## Coding Guidelines

Slurm follows the [Linux Kernel coding style](https://www.kernel.org/doc/html/latest/process/coding-style.html) with some modifications:

### Formatting

- **Indentation**: Tabs (8 spaces wide)
- **Line length**: 80 characters max
- **Braces**: K&R style

```c
if (condition) {
        do_something();
        do_more();
} else {
        do_other();
}
```

### Naming Conventions

- Functions: `lowercase_with_underscores`
- Macros: `UPPERCASE_WITH_UNDERSCORES`
- Types: `name_t` suffix for typedefs
- Global prefixes: `slurm_`, `slurmdb_`

### Comments

```c
/* Single line comment */

/*
 * Multi-line comment
 * continues here
 */

// C++ style also acceptable
```

### Error Messages

- Do not break error strings mid-sentence
- Split on format sequences, commas, or periods
- Enables easier grep searching

```c
// Good
error("Job %u failed to allocate resources on node %s",
      job_id, node_name);

// Bad - broken mid-sentence
error("Job %u failed to allocate "
      "resources on node %s", job_id, node_name);
```

---

## Testing

### Unit Tests (Check Framework)

```bash
# Build and run unit tests
cd testsuite/slurm_unit
make check
```

Location: `testsuite/slurm_unit/`

### Functional Tests (Expect)

```bash
# Run expect tests (requires running Slurm cluster)
cd testsuite/expect
./regression.py
```

Location: `testsuite/expect/`

### Python Tests

```bash
# Run Python tests
cd testsuite/python
pytest
```

Location: `testsuite/python/`

---

## Debugging

### Enable Debug Output

```bash
# In slurm.conf
SlurmctldDebug=debug5
SlurmdDebug=debug5
```

### GDB Debugging

```bash
# Attach to running daemon
gdb -p $(pidof slurmctld)

# Debug from start
gdb --args slurmctld -D -vvvvv
```

### Log Locations

- slurmctld: `/var/log/slurmctld.log`
- slurmd: `/var/log/slurmd.log`
- slurmdbd: `/var/log/slurmdbd.log`

---

## Plugin Development

### Plugin Structure

```c
// myplugin.c
#include "slurm/slurm.h"

const char plugin_name[] = "My Plugin";
const char plugin_type[] = "select/myplugin";
const uint32_t plugin_version = SLURM_VERSION_NUMBER;

// Plugin initialization
extern int init(void)
{
    return SLURM_SUCCESS;
}

// Plugin cleanup
extern int fini(void)
{
    return SLURM_SUCCESS;
}

// Plugin-specific functions...
```

### Building Plugins

Plugins are built automatically with the main build. For out-of-tree:

```bash
gcc -shared -fPIC -o myplugin.so myplugin.c \
    -I/usr/local/include/slurm
```

---

## Common Development Tasks

### Adding a New CLI Option

1. Edit `src/s*/opts.c` - add option parsing
2. Update man page in `doc/man/man1/`
3. Add tests in `testsuite/`

### Adding a New RPC

1. Define message type in `src/common/slurm_protocol_defs.h`
2. Add pack/unpack in `src/common/slurm_protocol_pack.c`
3. Add handler in relevant daemon
4. Update protocol version if needed

### Adding a New Plugin Type

1. Define interface in `src/interfaces/`
2. Create plugin directory in `src/plugins/`
3. Add to `configure.ac` and `Makefile.am`
4. Document in `doc/html/`

---

## Contribution Process

1. **Issue Tracker**: https://support.schedmd.com/
2. **No Pull Requests** - Submit patches via issue tracker
3. **Patch Format**: Use `git format-patch`
4. **Target Branch**:
   - New features → `master`
   - Bug fixes → Most recent stable

### Commit Message Format

```
Changelog: Brief description of change

Detailed explanation of what and why.
```

### Submission Checklist

- [ ] Code follows style guidelines
- [ ] Tests pass
- [ ] Documentation updated
- [ ] Commit message includes Changelog trailer
- [ ] Patches apply cleanly
- [ ] No generated files included

---

## Resources

### Documentation

- Official: https://slurm.schedmd.com/
- Admin Guide: https://slurm.schedmd.com/quickstart_admin.html
- API Reference: https://slurm.schedmd.com/api.html

### Source References

| Topic | Location |
|-------|----------|
| Main headers | `slurm/slurm.h`, `slurm/slurmdb.h` |
| Protocol | `src/common/slurm_protocol_*.h` |
| Job management | `src/slurmctld/job_mgr.c` |
| Node management | `src/slurmctld/node_mgr.c` |
| Scheduling | `src/slurmctld/job_scheduler.c` |
| Plugin API | `src/interfaces/` |

### Community

- Mailing list: slurm-users@lists.schedmd.com
- Issue tracker: https://support.schedmd.com/
