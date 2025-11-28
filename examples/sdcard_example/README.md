# Blackbox Logger - SD Card Example

This example demonstrates using the blackbox logger library with an SD card for high-throughput data logging, ideal for flight controllers and data acquisition systems.

## Overview

SD card storage is suitable for:
- Large log files (GB of storage available)
- High-speed data logging (50+ Hz)
- Long-duration recording sessions
- Easy log retrieval (remove SD card and read on PC)
- Flight data recorders / black boxes

## Hardware Requirements

- ESP32 board
- SD card module (SPI interface)
- MicroSD card (FAT32 formatted)
- Jumper wires

### Default Pin Configuration

| SD Card Pin | ESP32 GPIO | Description |
|-------------|------------|-------------|
| MOSI | GPIO 23 | Master Out Slave In |
| MISO | GPIO 19 | Master In Slave Out |
| CLK | GPIO 18 | SPI Clock |
| CS | GPIO 5 | Chip Select |
| VCC | 3.3V | Power |
| GND | GND | Ground |

> **Note:** Pins can be changed in `main.c`

## Features Demonstrated

- SD card initialization via SPI
- High-frequency logging (50Hz flight data)
- Multiple concurrent logging tasks
- Message rate calculation
- Log file listing
- Automatic file rotation
- Statistics monitoring

## Project Structure

```
sdcard_example/
├── CMakeLists.txt          # Project CMake configuration
├── sdkconfig.defaults      # Default SDK configuration
└── main/
    ├── CMakeLists.txt      # Main component CMake
    ├── idf_component.yml   # Component dependencies
    └── main.c              # Application source code
```

## Configuration

### Logger Configuration

```c
blackbox_config_t config;
blackbox_get_default_config(&config);

config.root_path = "/sdcard/logs";      // Log directory on SD card
config.file_prefix = "flight";          // Files: flight001.blackbox, etc.
config.file_size_limit = 512 * 1024;    // 512KB per file
config.buffer_size = 32 * 1024;         // 32KB ring buffer
config.flush_interval_ms = 100;         // Fast flush for high-rate data
config.min_level = BLACKBOX_LOG_LEVEL_DEBUG;
config.console_output = true;
config.file_output = true;
config.encrypt = false;                 // No encryption
```

## Simulated Data

The example simulates two data sources:

### Flight Data Task (50Hz)
- Roll, pitch, yaw angles
- Altitude
- Throttle position
- Battery voltage

### GPS Task (1Hz)
- Latitude/Longitude
- Speed
- Satellite count

## Building and Flashing

1. Set up ESP-IDF environment:
   ```bash
   . $IDF_PATH/export.sh
   ```

2. Navigate to the example directory:
   ```bash
   cd examples/sdcard_example
   ```

3. (Optional) Configure pins in menuconfig:
   ```bash
   idf.py menuconfig
   ```

4. Build the project:
   ```bash
   idf.py build
   ```

5. Flash and monitor:
   ```bash
   idf.py -p /dev/ttyUSB0 flash monitor
   ```

## Expected Output

```
I (325) SDCARD_EXAMPLE: === Blackbox Logger SD Card Example ===
I (335) SDCARD_EXAMPLE: Initializing SD card via SPI...
I (385) SDCARD_EXAMPLE: SD card mounted successfully!
Name: SD32G
Type: SDHC/SDXC
Speed: 20 MHz
Size: 30436MB
I (405) SDCARD_EXAMPLE: Created directory: /sdcard/logs
I (415) BLACKBOX_LOG: Logger initialized: path=/sdcard/logs, encrypt=0, buffer=32KB
I (425) SDCARD_EXAMPLE: === Flight Session Started ===
I (435) BLACKBOX_LOG: Created log file: /sdcard/logs/flight001.blackbox
D (455) FLIGHT: F:1 R:0.12 P:-0.05 Y:0.00 A:100.2 T:1498 V:12.59
D (475) FLIGHT: F:2 R:0.15 P:-0.08 Y:0.01 A:100.5 T:1502 V:12.59
...
I (1435) GPS: Lat:37.774912 Lon:-122.419388 Spd:2.3 Sat:9
...
I (5435) SDCARD_EXAMPLE: === Logger Statistics ===
I (5435) SDCARD_EXAMPLE: Messages: logged=267, dropped=0
I (5435) SDCARD_EXAMPLE: Storage: written=28 KB, files=1
I (5435) SDCARD_EXAMPLE: Message rate: 53.4 msg/sec
```

## Log File Location

Log files are stored at `/sdcard/logs/` with names like:
- `flight001.blackbox`
- `flight002.blackbox`
- etc.

## Decoding Log Files

1. Remove the SD card from the device
2. Insert into PC card reader
3. Copy the `.blackbox` files
4. Decode using the Python tool:

```bash
python ../../tools/blackbox_decoder.py flight001.blackbox --stats

# Export to CSV for analysis
python ../../tools/blackbox_decoder.py flight001.blackbox --format csv --output flight_data.csv
```

## Performance Tuning

### For Higher Throughput
```c
config.buffer_size = 64 * 1024;     // Larger buffer
config.flush_interval_ms = 50;       // More frequent flushes
```

### For Lower CPU Usage
```c
config.buffer_size = 32 * 1024;     // Standard buffer
config.flush_interval_ms = 200;      // Less frequent flushes
config.console_output = false;       // Disable console mirroring
```

### For Longer Recording Sessions
```c
config.file_size_limit = 2 * 1024 * 1024;  // 2MB per file
config.min_level = BLACKBOX_LOG_LEVEL_INFO; // Skip DEBUG/VERBOSE
```

## Troubleshooting

### SD card not detected
- Check wiring connections
- Verify SD card is FAT32 formatted
- Try a different SD card
- Check SPI pins in code match your wiring

### Messages being dropped
- Increase `buffer_size`
- Decrease logging frequency
- Reduce `flush_interval_ms`
- Disable `console_output`

### Slow write speed
- Use a faster SD card (Class 10 or better)
- Increase SPI frequency (if supported)
- Use larger write chunks

### File system corruption
- Always flush before power off
- Use `blackbox_deinit()` for clean shutdown
- Consider using wear-leveling SD cards

## Wiring Diagram

```
ESP32                SD Card Module
┌─────────┐         ┌─────────────┐
│    3.3V ├─────────┤ VCC         │
│     GND ├─────────┤ GND         │
│  GPIO23 ├─────────┤ MOSI        │
│  GPIO19 ├─────────┤ MISO        │
│  GPIO18 ├─────────┤ CLK         │
│   GPIO5 ├─────────┤ CS          │
└─────────┘         └─────────────┘
```

## Memory Usage

- Ring buffer: 32KB (configurable)
- Stack for writer task: 4KB
- Stack for flight_data task: 4KB
- Stack for gps task: 4KB

## License

See the main project LICENSE file.
