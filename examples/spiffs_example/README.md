# Blackbox Logger - SPIFFS Example

This example demonstrates using the blackbox logger library with SPIFFS (SPI Flash File System) for storing log files on internal flash memory.

## Overview

SPIFFS is suitable for:
- Small to medium log files (limited by flash partition size)
- Applications where SD card hardware is not available
- Quick prototyping and testing
- Wear-leveling is handled by the filesystem

## Hardware Requirements

- Any ESP32 board (no external hardware needed)
- USB cable for flashing and monitoring

## Features Demonstrated

- SPIFFS filesystem initialization and mounting
- Automatic formatting if mount fails
- Simulated sensor readings (temperature/humidity)
- Logging at different levels based on conditions
- SPIFFS space monitoring
- Logger statistics reporting

## Project Structure

```
spiffs_example/
├── CMakeLists.txt          # Project CMake configuration
├── partitions.csv          # Custom partition table with SPIFFS
├── sdkconfig.defaults      # Default SDK configuration
└── main/
    ├── CMakeLists.txt      # Main component CMake
    ├── idf_component.yml   # Component dependencies
    └── main.c              # Application source code
```

## Configuration

### Partition Table

The example uses a custom partition table (`partitions.csv`) with:
- 1MB for the application
- 512KB for SPIFFS storage

```csv
# Name,   Type, SubType, Offset,   Size, Flags
nvs,      data, nvs,     0x9000,   0x6000,
phy_init, data, phy,     0xf000,   0x1000,
factory,  app,  factory, 0x10000,  1M,
spiffs,   data, spiffs,  ,         512K,
```

### Logger Configuration

```c
blackbox_config_t config;
blackbox_get_default_config(&config);

config.root_path = "/spiffs/logs";      // Log directory in SPIFFS
config.file_prefix = "sensor";          // Files: sensor001.blackbox, etc.
config.file_size_limit = 64 * 1024;     // 64KB per file (smaller for flash)
config.buffer_size = 16 * 1024;         // 16KB ring buffer (minimum)
config.flush_interval_ms = 500;         // Flush every 500ms
config.min_level = BLACKBOX_LOG_LEVEL_DEBUG;
config.console_output = true;
config.file_output = true;
config.encrypt = false;                 // No encryption
```

## Building and Flashing

1. Set up ESP-IDF environment:
   ```bash
   . $IDF_PATH/export.sh
   ```

2. Navigate to the example directory:
   ```bash
   cd examples/spiffs_example
   ```

3. Build the project:
   ```bash
   idf.py build
   ```

4. Flash and monitor:
   ```bash
   idf.py -p /dev/ttyUSB0 flash monitor
   ```

## Expected Output

```
I (325) SPIFFS_EXAMPLE: === Blackbox Logger SPIFFS Example ===
I (335) SPIFFS_EXAMPLE: Initializing SPIFFS...
I (385) SPIFFS_EXAMPLE: SPIFFS: Total: 956561 bytes, Used: 0 bytes
I (395) BLACKBOX_LOG: Logger initialized: path=/spiffs/logs, encrypt=0, buffer=16KB
I (405) SPIFFS_EXAMPLE: Blackbox logger initialized successfully
I (415) BLACKBOX_LOG: Created log file: /spiffs/logs/sensor001.blackbox
I (425) SENSOR: Reading #1: Temp=25.12°C, Humidity=50.34%
I (1425) SENSOR: Reading #2: Temp=25.08°C, Humidity=50.21%
...
```

## Log File Location

Log files are stored at `/spiffs/logs/` with names like:
- `sensor001.blackbox`
- `sensor002.blackbox`
- etc.

## Decoding Log Files

To decode the log files, copy them from the device and use the decoder tool:

```bash
# Copy files from device (requires esptool or similar)
# Then decode:
python ../../tools/blackbox_decoder.py sensor001.blackbox
```

## Troubleshooting

### SPIFFS mount fails
- Ensure the partition table is flashed: `idf.py partition-table-flash`
- Try erasing flash: `idf.py erase-flash`

### "No space left" errors
- SPIFFS partition is full
- Increase partition size or reduce `file_size_limit`
- Implement log rotation/cleanup

### Slow write performance
- SPIFFS is slower than SD card
- Consider increasing `flush_interval_ms`

## Memory Usage

- SPIFFS partition: 512KB
- Ring buffer: 16KB (configurable)
- Stack for writer task: 4KB

## License

See the main project LICENSE file.
