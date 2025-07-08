# eBPF System Monitor

Real-time system monitoring tool using eBPF to track process creation and file access events with Prometheus metrics support.

## Overview

This project consists of two main components:
- **eBPF Kernel Program** (`ebpf-probe.c`) - Runs in kernel space to capture system events
- **Python Controller** (`ebpf-runner.py`) - Manages the eBPF program and processes events in userspace

## Technical Details

### eBPF Kernel Program (ebpf-probe.c)

The kernel-space eBPF program uses **kprobes** to hook into critical kernel functions:

#### Monitored System Calls:
- **`sys_clone`** - Captures process creation events
- **`do_sys_openat2`** - Captures file access operations

#### Data Structures:
```c
struct clone_data_t {
    u32 pid;        // Process ID
    u32 ppid;       // Parent Process ID  
    char comm[TASK_COMM_LEN];  // Process name
};

struct open_data_t {
    u32 pid;        // Process ID
    u64 timestamp;  // Event timestamp (nanoseconds)
    char comm[TASK_COMM_LEN];  // Process name
    char filename[NAME_MAX];   // Accessed file path
};
```

#### Event Delivery:
- Uses `BPF_PERF_OUTPUT` buffers for efficient kernel-to-userspace communication
- Events are delivered in real-time via perf ring buffers

### Python Controller (ebpf-runner.py)

The userspace component leverages the **BCC (BPF Compiler Collection)** framework:

#### Key Features:
- **Event Processing** - Handles events from kernel perf buffers
- **Real-time Display** - Prints formatted events to console
- **Metrics Collection** - Exposes Prometheus metrics on port 3000
- **Error Handling** - Graceful shutdown on interruption

#### Prometheus Metrics:
- `sys_clone_calls_total` - Counter for process creation events
- `sys_openat_calls_total` - Counter for file access events

## Prerequisites

### System Requirements:
- **Linux Kernel** 4.1+ (with eBPF support)
- **Root privileges** (required for eBPF program loading)

### Dependencies:
- **BCC (BPF Compiler Collection)**
- **Python 3.6+**
- **prometheus_client** Python package

## Installation

### 1. Install BCC

#### Ubuntu/Debian:
```bash
sudo apt update
sudo apt install bpfcc-tools linux-headers-$(uname -r)
```

#### CentOS/RHEL/Fedora:
```bash
sudo yum install bcc-tools kernel-devel
# or for newer versions:
sudo dnf install bcc-tools kernel-devel
```

### 2. Install Python Dependencies
```bash
pip3 install prometheus_client
# or
pip3 install -r requirements.txt  # if you create one
```

### 3. Clone Repository
```bash
git clone https://github.com/Ahmedwagdymohy/ebpf-system-monitor
cd ebpf-system-monitor
```

## Usage

### Basic Monitoring

Run the system monitor (requires root privileges):

```bash
sudo python3 eBPF/ebpf-runner.py
```

### Expected Output:
```
Monitoring sys_clone and file open events... Press Ctrl+C to exit.
Process bash (PID: 12345, PPID: 12344) called sys_clone
[1234567890.123456] Process ls (PID: 12346) opened file: /bin/ls
[1234567890.234567] Process cat (PID: 12347) opened file: /etc/passwd
```

### Prometheus Metrics

Access metrics at: `http://localhost:3000/metrics`

Example metrics output:
```
# HELP sys_clone_calls_total Number of sys_clone calls
# TYPE sys_clone_calls_total counter
sys_clone_calls_total 42.0

# HELP sys_openat_calls_total Number of sys_openat calls  
# TYPE sys_openat_calls_total counter
sys_openat_calls_total 156.0
```

## Configuration

### Customizing Monitoring

To modify what events are captured, edit `eBPF/ebpf-probe.c`:

1. **Add new kprobes** for different system calls
2. **Modify data structures** to capture additional fields
3. **Update event handlers** in `ebpf-runner.py`

### Changing Metrics Port

Modify the port in `ebpf-runner.py`:
```python
start_http_server(3000)  # Change to desired port
```

## Use Cases

- **Security Monitoring** - Track process spawning and file access
- **Performance Analysis** - Monitor system call patterns
- **Compliance Auditing** - Log file access for regulatory requirements
- **Development Debugging** - Understand application behavior at kernel level

## Troubleshooting

### Common Issues:

1. **Permission Denied**
   ```bash
   sudo python3 eBPF/ebpf-runner.py
   ```

2. **BCC Not Found**
   - Ensure BCC is properly installed
   - Check kernel headers are available

3. **eBPF Program Load Failed**
   - Verify kernel supports eBPF (4.1+)
   - Check for sufficient privileges

### Debug Mode:
Add verbose output by modifying the event handlers to print additional information.

## Performance Impact

- **Minimal overhead** - eBPF runs in kernel space with optimized execution
- **Efficient filtering** - Events processed only when needed
- **Scalable** - Handles high-frequency system calls efficiently

## Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Security Considerations

- Requires root privileges for eBPF program loading
- Monitor output may contain sensitive file paths
- Consider log rotation for production deployments
- Review captured data for compliance requirements
