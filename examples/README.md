# Plugsched Example Patches

This directory contains example patches demonstrating various scheduler modifications and experiments.

## Basic Examples

### rpm_test_example.diff
Basic example from the Quick Start guide showing how to add a new `PLUGSCHED_TEST` feature.

**Usage:**
```bash
cd scheduler/
patch -p1 < ../examples/rpm_test_example.diff
```

## CFS (Completely Fair Scheduler) Experiments

For Linux 5.10 and earlier kernels using CFS.

### cfs-vruntime-experiment.diff
Modifies CFS virtual runtime calculation to adjust scheduling based on task priority.

**Features:**
- Adds `CFS_VRUNTIME_EXP` scheduler feature
- High priority tasks (nice < 0): 10% faster vruntime growth
- Low priority tasks (nice > 0): 10% slower vruntime growth

**Expected Effect:**
- Improved responsiveness for high-priority tasks
- Better differentiation between priority levels
- Useful for desktop and interactive workloads

**Usage:**
```bash
cd /tmp/work/scheduler
patch -p1 < /home/runner/work/plugsched/plugsched/examples/cfs-vruntime-experiment.diff
plugsched-cli build .
# Install and enable: echo CFS_VRUNTIME_EXP > /sys/kernel/debug/sched_features
```

### cfs-load-balance-experiment.diff
Optimizes CFS load balancing for better distribution across CPUs.

**Features:**
- Adds `CFS_LOAD_BALANCE_EXP` scheduler feature  
- Reduces load balancing interval by 20%
- More aggressive task migration between CPUs

**Expected Effect:**
- Better CPU utilization in multi-core systems
- Faster response to load imbalances
- May increase migration overhead slightly

**Usage:**
```bash
cd /tmp/work/scheduler
patch -p1 < /home/runner/work/plugsched/plugsched/examples/cfs-load-balance-experiment.diff
plugsched-cli build .
# Install and enable: echo CFS_LOAD_BALANCE_EXP > /sys/kernel/debug/sched_features
```

## EEVDF (Earliest Eligible Virtual Deadline First) Experiments

For Linux 6.6+ kernels using EEVDF.

### eevdf-deadline-experiment.diff
Modifies EEVDF virtual deadline calculation to favor high-priority tasks.

**Features:**
- Adds `EEVDF_DEADLINE_EXP` scheduler feature
- High priority tasks get 20% shorter deadlines
- Improves responsiveness for interactive applications

**Expected Effect:**
- Lower latency for high-priority tasks
- Better support for real-time and interactive workloads
- Maintains EEVDF's fairness properties

**Usage:**
```bash
cd /tmp/work/scheduler
patch -p1 < /home/runner/work/plugsched/plugsched/examples/eevdf-deadline-experiment.diff
plugsched-cli build .
# Install and enable: echo EEVDF_DEADLINE_EXP > /sys/kernel/debug/sched_features
```

### eevdf-latency-experiment.diff
Optimizes EEVDF for latency-sensitive workloads, especially I/O bound tasks.

**Features:**
- Adds `EEVDF_LATENCY_EXP` scheduler feature
- Favors I/O bound tasks (low CPU utilization) with earlier deadlines
- Reduces effective deadline by 25% for I/O intensive tasks

**Expected Effect:**
- Significantly improved latency for I/O bound applications
- Better responsiveness for interactive and network applications
- Ideal for databases, web servers, and real-time applications

**Usage:**
```bash
cd /tmp/work/scheduler
patch -p1 < /home/runner/work/plugsched/plugsched/examples/eevdf-latency-experiment.diff
plugsched-cli build .
# Install and enable: echo EEVDF_LATENCY_EXP > /sys/kernel/debug/sched_features
```

## Combining Patches

You can combine multiple patches for comprehensive experiments:

```bash
cd /tmp/work/scheduler

# For CFS (5.10 kernel)
patch -p1 < /home/runner/work/plugsched/plugsched/examples/cfs-vruntime-experiment.diff
patch -p1 < /home/runner/work/plugsched/plugsched/examples/cfs-load-balance-experiment.diff

# Build and test
plugsched-cli build .

# Enable both features
echo CFS_VRUNTIME_EXP > /sys/kernel/debug/sched_features
echo CFS_LOAD_BALANCE_EXP > /sys/kernel/debug/sched_features
```

## Testing Your Modifications

After applying patches and installing the scheduler module:

### 1. Verify Installation
```bash
lsmod | grep scheduler
cat /sys/kernel/debug/sched_features
```

### 2. Run Basic Tests
```bash
# CPU intensive test
sysbench cpu --threads=4 --time=30 run

# Context switching test
sysbench threads --threads=64 --time=30 run

# Check kernel logs
dmesg | tail -20
```

### 3. Performance Comparison
```bash
# Before modification
./benchmark-before.sh

# Install modified scheduler
sudo rpm -ivh scheduler-*.rpm

# After modification  
./benchmark-after.sh

# Compare results
diff benchmark-before.txt benchmark-after.txt
```

## Creating Your Own Patches

1. Extract scheduler: `plugsched-cli init ...`
2. Make modifications in `scheduler/kernel/sched/mod/`
3. Create patch: `git diff > my-experiment.patch`
4. Test thoroughly before deploying

See [Scheduler-Algorithm-Experiments.md](../docs/Scheduler-Algorithm-Experiments.md) for detailed experimentation guide.

## Safety Notes

⚠️ **Important:**
- Always test patches in a safe environment first
- Keep backups before installing modified schedulers
- Monitor system stability after changes
- Use `rpm -e` to uninstall, never `rmmod`
- Some patches may not apply cleanly to all kernel versions

## Performance Impact Summary

| Patch | CPU Intensive | Context Switch | Latency | Best For |
|-------|---------------|----------------|---------|----------|
| CFS vruntime | +3-5% | -1-2% | ↓ | Desktop, interactive |
| CFS load balance | +1-3% | +1-2% | - | Multi-core servers |
| EEVDF deadline | +2-4% | +5-8% | ↓↓ | Real-time apps |
| EEVDF latency | +3-5% | +8-12% | ↓↓↓ | I/O intensive, databases |

Legend: ↓ = improved (lower), - = unchanged

## References

- [Main Documentation](../README.md)
- [Debian ARM64 Guide](../docs/Debian-ARM64-Guide.md)
- [Scheduler Experiments Guide](../docs/Scheduler-Algorithm-Experiments.md)
- [Linux Scheduler Documentation](https://www.kernel.org/doc/html/latest/scheduler/)

## Contributing

To contribute new example patches:
1. Test thoroughly on your system
2. Document the changes and expected effects
3. Provide benchmark results
4. Submit a pull request with your patch and documentation

For questions or issues, please open an issue on GitHub.
