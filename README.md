# Linux System Monitor

A real-time terminal-based system monitoring tool written in pure Bash that provides comprehensive insights into CPU, memory, swap, disk, and network performance with dynamic visualizations.

## Overview

This system monitor leverages Linux's `/proc` filesystem and system utilities to gather and display live performance metrics in a color-coded, interactive terminal interface. The tool uses ANSI escape sequences for advanced terminal control, enabling a responsive dashboard that updates in real-time while maintaining minimal system overhead.
[SS_1.png]

## Technical Architecture

### Data Sources

The monitor directly interfaces with Linux kernel data structures through the `/proc` filesystem:

- **CPU Statistics**: `/proc/stat` - Provides per-core CPU time accounting across multiple states (user, nice, system, idle, iowait, irq, softirq, steal, guest)
- **CPU Frequency**: `/proc/cpuinfo` - Extracts real-time clock speeds for each core
- **Memory Information**: `/proc/meminfo` - Tracks total, free, available, cached, and swap memory metrics
- **Network Statistics**: `/proc/net/dev` - Monitors bytes transmitted and received per network interface
- **Disk Usage**: `df` command - Calculates filesystem usage for root and home partitions
- **Disk Hardware**: `lsblk` - Identifies physical disk types (HDD/SSD/NVMe) and capacities

### Core Components

#### 1. CPU Monitoring

**Data Collection Algorithm**:
```
For each CPU core:
  - Read current busy time (user + nice + system + irq + softirq + steal + guest + guest_nice)
  - Read current idle time (idle + iowait)
  - Read current clock frequency from /proc/cpuinfo
  - Store as: "busy_time idle_time frequency"
```

**Usage Calculation**:
```
Δbusy = busy_current - busy_previous
Δidle = idle_current - idle_previous
Δtotal = Δbusy + Δidle
CPU% = (Δbusy / Δtotal) × 100
```

The monitor maintains two arrays (`cpu_current` and `cpu_prev`) to track CPU states between iterations, enabling accurate delta calculations. This differential approach accounts for varying time intervals and provides precise utilization metrics.

**Color Coding Logic**:
- 0-20%: Green (danger_color) - Low usage
- 20-40%: Light green (semi_danger_color)
- 40-60%: Yellow (okay_color)
- 60-80%: Orange (safe_color)
- 80-100%: Red (very_safe_color) - High usage

#### 2. Memory Management

**Tracked Metrics**:
- Total RAM capacity
- Used memory (calculated as Total - Available)
- Available memory for applications
- Free memory (actual unused RAM)
- Cache usage

**Display Strategy**:
The tool presents two progress bars:
1. Available memory percentage (relative to total)
2. Free memory percentage (relative to available)

This dual-bar approach distinguishes between memory that's truly free versus memory that's available through cache eviction.

#### 3. Swap Statistics

Monitors swap space usage with identical visualization logic as main memory:
- Total swap capacity
- Used swap space
- Cached swap (swap pages that are also in RAM)
- Free swap space

The color coding is inverted compared to CPU (lower usage is better for memory/swap).

#### 4. Disk Analysis

**Hardware Detection**:
- Uses `lsblk` to enumerate block devices
- Determines disk type through ROTA flag (rotational) and device naming conventions:
  - ROTA=1 → Hard Disk Drive (HDD)
  - `sda*` with ROTA=0 → Solid State Drive (SSD)
  - `nvme*` → NVMe SSD
- Identifies the root filesystem's backing device using `findmnt` and `lsblk -no PKNAME`

**Space Tracking**:
Monitors both root (`/`) and home (`/home`) partitions separately, displaying total, used, and free space in gigabytes.

#### 5. Network Monitoring

**Interface Detection**:
Scans `/proc/net/dev` while filtering out:
- Loopback interfaces (`lo*`)
- LXC bridge interfaces (`lxcbr*`)

**Rate Calculation**:
```
For each interface:
  bytes_delta = bytes_current - bytes_previous
  rate = bytes_delta / time_interval
  rate_KB/s = rate / 1000
```

**Status Tracking**:
- Reads interface operational state from `/sys/class/net/{interface}/operstate`
- Distinguishes between `up` (active) and `down` (inactive) interfaces
- Color codes: Green for active connections, red for disconnected

**Interface Type Identification**:
- `wl*` → Wireless interfaces
- `enp*` → Ethernet interfaces

### Terminal Control

#### ANSI Escape Sequences

The monitor employs sophisticated terminal manipulation:

**Buffer Management**:
- `\e[?1049h` - Enable alternate screen buffer (preserves original terminal content)
- `\e[?1049l` - Restore main screen buffer on exit

**Cursor Control**:
- `\e[?25l` - Hide cursor during updates (prevents flicker)
- `\e[?25h` - Restore cursor visibility on exit
- `\e7` - Save cursor position
- `\e8` - Restore cursor position
- `\e[{line};{col}H` - Absolute cursor positioning

**Dynamic Layout**:
The monitor calculates component visibility based on terminal dimensions:
- Minimum 7 lines: CPU stats only
- Minimum 15 lines: CPU + Memory
- Minimum 22 lines: CPU + Memory + Swap
- Minimum 30 lines: CPU + Memory + Swap + Disk
- Minimum 35 lines + 75 cols: All stats including Network

**Progress Bar Rendering**:
```
Available width = terminal_columns - fixed_elements_width
Bar length = (usage_percentage × available_width) / 100
```

Each bar dynamically scales to terminal width while preserving space for labels and percentages.

### Data Flow Architecture

```
Initialization:
  1. Read initial CPU state
  2. Count CPU cores
  3. Detect disk hardware
  4. Set up terminal (alternate buffer, hide cursor)

Main Loop (every sleep_delay seconds):
  1. Copy previous state (CPU, network)
  2. Read current state (CPU, network, memory, disk)
  3. Calculate deltas and rates
  4. Render visualizations
  5. Sleep for configured interval
```

### Performance Optimization

**Efficient Parsing**:
- Uses Bash built-in `read` for file parsing (no external process spawning)
- Employs `mapfile` for array population in single operations
- Minimizes subshell creation through process substitution

**Minimal External Commands**:
Only invokes external utilities when necessary:
- `awk` for CPU frequency and disk stats
- `df` for filesystem usage
- `lsblk` and `findmnt` for disk discovery (once at startup)
- `cat` for interface state (per iteration, but minimal overhead)

**Delta-based Updates**:
Instead of absolute measurements, the tool calculates differences between snapshots, providing accurate metrics regardless of collection interval.

### Configuration

**Command-line Options**:
- `-s <seconds>` - Set update interval (default must be provided at runtime)

**Color Scheme**:
All colors are defined via 256-color ANSI codes (`\e[38;5;{color}m` for foreground, `\e[48;5;{color}m` for background), ensuring broad terminal compatibility.

### Error Handling

**Terminal Detection**:
- Validates terminal size via `stty size </dev/tty` with error suppression
- Falls back gracefully if terminal operations fail
- Returns early from functions if terminal is unavailable

**Signal Handling**:
- `trap deinit_term exit` - Ensures proper cleanup (restore cursor, disable alternate buffer)
- `trap init_term WINCH` - Reinitializes display on terminal resize

**Window Size Awareness**:
- `shopt -s checkwinsize` - Enables automatic `LINES` and `COLUMNS` updates
- Dummy command `(:)` forces initial window size detection

## Technical Requirements

- **Shell**: Bash 4.0+ (for associative arrays)
- **Kernel**: Linux with `/proc` filesystem
- **Terminal**: ANSI-compatible terminal with 256-color support
- **Privileges**: Standard user (no root required for system monitoring)

## Usage

```bash
./system_monitor.sh -s <update_interval_seconds>
```

Example:
```bash
./system_monitor.sh -s 2  # Updates every 2 seconds
```

## Implementation Details

### Variable Declarations

**Arrays**:
- `cpu_current[]` - Current CPU state per core
- `cpu_prev[]` - Previous CPU state for delta calculation
- `network_current[]` - Current network statistics per interface
- `network_previous[]` - Previous network state

**Associative Arrays**:
- `disk_info[]` - Disk hardware information indexed by disk number
- `mem_values[]` - Memory metrics indexed by type (total, free, cached, etc.)

### Key Functions

- `init_systems()` - Initializes disk detection, CPU state, and core count
- `copy_and_read_data()` - Orchestrates state copying and data collection
- `visualise_data()` - Master rendering function that calls all visualization components
- `print_bar_chart_generic()` - Reusable progress bar renderer with dynamic scaling
- `print-title()` - Renders section headers with background color fills

## License

This project is a system monitoring utility distributed as-is for educational and practical use.
