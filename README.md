# Dragonfly Testing Framework with AFL++

This framework is designed for testing the [Dragonfly](https://github.com/dragonflydb/dragonfly) server using American Fuzzy Lop (AFL++). Dragonfly is a modern replacement for Redis and Memcached with high performance.

## Overview

df-afl is a comprehensive tool for fuzz testing Dragonfly that:
- Generates random Redis commands
- Uses AFL++ for systematic testing
- Allows focusing on specific commands
- Uses dictionaries for more effective testing

## Prerequisites

### 1. Installing AFL++

AFL++ must be installed separately:

```bash
# Installation from package manager (Ubuntu/Debian)
sudo apt update
sudo apt install afl++

# Or build from source
git clone https://github.com/AFLplusplus/AFLplusplus
cd AFLplusplus
make distrib
sudo make install
```

More details:
- [AFL++ Documentation](https://aflplus.plus/)
- [AFL++ Source code](https://github.com/AFLplusplus/AFLplusplus)

### 2. Cloning and Building Dragonfly

```bash
# Clone the repository
git clone https://github.com/dragonflydb/dragonfly.git
cd dragonfly

# Install dependencies and build (see CONTRIBUTING.md)
./helio/blaze.sh
cd build-dbg
ninja dragonfly
```

Detailed build instructions: [CONTRIBUTING.md](https://github.com/dragonflydb/dragonfly/blob/main/CONTRIBUTING.md)

### 3. Building the Traffic Replay Tool

```bash
cd dragonfly/tools/replay
go build
```

This will create the `traffic-replay` executable.

## Usage

### 1. Running Dragonfly with Traffic Recording

For debugging, it's recommended to run Dragonfly with traffic recording enabled:

```bash
# Create directory for traffic
mkdir ~/traffic

# Start Dragonfly
./dragonfly --dbfilename= --logtostderr

# In a separate terminal, activate traffic recording
redis-cli DEBUG TRAFFIC ~/traffic/traffic
```

### 2. System Preparation for Testing

For optimal AFL++ performance, configure the system:

```bash
sudo su -
echo core >/proc/sys/kernel/core_pattern
cd /sys/devices/system/cpu
echo performance | tee cpu*/cpufreq/scaling_governor
```

You can also perform the same preparation automatically:

```bash
./scripts/prepare_system.sh
```

(The script will ask for sudo privileges if not run as root.)

### 3. Running Tests

```bash
# Navigate to the df-afl directory
cd df-afl

# Start fuzzing
./run_afl_fuzzing.sh
```

### 4. Reproducing Issues

If bugs are found, you can replay the traffic:

```bash
./traffic-replay -ignore-parse-errors run traffic/traffic*
```

## Configuration

### Environment Variables

#### DICT_MIX_RATIO

Controls the ratio between dictionary values and generated values (0-1):

```bash
export DICT_MIX_RATIO=0.7  # 70% dictionary values, 30% random
export DICT_MIX_RATIO=0.0  # Only random values
export DICT_MIX_RATIO=1.0  # Only dictionary values (if available)
```

Default: `0.5` (50/50)

#### REDIS_FOCUS_COMMANDS

Allows focusing on specific Redis commands:

```bash
# Focus on a single command (30% probability)
export REDIS_FOCUS_COMMANDS="SET"

# Focus on multiple commands (50% probability for any of them)
export REDIS_FOCUS_COMMANDS="SET,GET,HSET,SADD"

# Disable focus commands
unset REDIS_FOCUS_COMMANDS
```

#### REDIS_EXCLUDE_COMMANDS

Excludes potentially dangerous Redis/Dragonfly commands from fuzzing (to avoid accidental data loss or server shutdown).

By default, three high-risk commands are disabled:

```bash
SHUTDOWN  # Stops the server
FLUSHDB   # Removes all keys from the current database
FLUSHALL  # Removes all keys from all databases
```

Configure the exclusion list via an environment variable:

```bash
# Extend the exclusion list (comma-separated)
export REDIS_EXCLUDE_COMMANDS="SHUTDOWN,FLUSHDB,FLUSHALL,SAVE"

# Disable exclusions completely (NOT recommended on a production server)
export REDIS_EXCLUDE_COMMANDS=""

# Or simply unset the variable
unset REDIS_EXCLUDE_COMMANDS
```

#### Other Variables

```bash
export REDIS_HOST="127.0.0.1"        # Redis/Dragonfly host
export REDIS_PORT="6379"             # Redis/Dragonfly port
export OUTPUT_DIR="./output"         # AFL++ output directory
export INPUT_DIR="./input"           # Initial test cases directory
export MAX_COMMANDS="30"             # Maximum commands per test
```

## Project Structure

- `redis_fuzzer.py` - Main fuzzer with AFL++ support
- `redis_commands.py` - Redis command definitions and data generators
- `redis_dict_generator.py` - Dictionary generator for AFL++
- `run_afl_fuzzing.sh` - Test execution script
- `scripts/prepare_system.sh` - System preparation helper script
- `input/` - Initial test cases
- `output/` - AFL++ results (crashes, hangs, queue)

## Features

### Generation Strategies

1. **Random generation** - Completely random commands and arguments
2. **Dictionary generation** - Using predefined values
3. **Mixed strategy** - Combination of both approaches
4. **Focus testing** - Concentration on specific commands

### Data Types

Various data types are generated:
- Regular strings
- Special characters
- Escape sequences
- Binary data
- JSON structures
- Vector data

## Development

### Adding New Commands

Edit `REDIS_COMMANDS` in `redis_commands.py`:

```python
"MYCOMMAND": {
    "args": ["key", "value"], 
    "optional_args": ["option1", "option2"]
}
```

### New Data Types

Add to `DATA_TYPES`:

```python
"mytype": lambda: generate_my_custom_data()
```
