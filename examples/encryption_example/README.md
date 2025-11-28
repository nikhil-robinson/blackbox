# Blackbox Logger - Encryption Example

This example demonstrates using the blackbox logger library with AES-256-CTR encryption enabled for secure logging of sensitive data.

## Overview

Encrypted logging is suitable for:
- Protecting proprietary data and algorithms
- Storing sensitive user/personal information
- Compliance with data protection requirements (GDPR, etc.)
- Tamper-proof audit logs
- Secure flight data recording

## Hardware Requirements

- ESP32 board
- SD card module (SPI interface)
- MicroSD card (FAT32 formatted)
- Jumper wires

### Pin Configuration

Same as SD Card example:
| SD Card Pin | ESP32 GPIO |
|-------------|------------|
| MOSI | GPIO 23 |
| MISO | GPIO 19 |
| CLK | GPIO 18 |
| CS | GPIO 5 |

## Features Demonstrated

- AES-256-CTR encryption
- Secure key handling (with warnings)
- Encrypted file output
- Plaintext console output (for debugging)
- Sensitive data logging simulation
- Security event logging

## Project Structure

```
encryption_example/
├── CMakeLists.txt          # Project CMake configuration
├── sdkconfig.defaults      # Default SDK configuration (enables mbedTLS)
└── main/
    ├── CMakeLists.txt      # Main component CMake
    ├── idf_component.yml   # Component dependencies
    └── main.c              # Application source code
```

## Encryption Details

### Algorithm
- **Cipher:** AES-256-CTR (Counter Mode)
- **Key Size:** 256 bits (32 bytes)
- **IV Size:** 128 bits (16 bytes, randomly generated per file)

### What's Encrypted
- All log packet data after the file header
- Message payloads, timestamps, levels, etc.

### What's NOT Encrypted
- File header (to identify file type)
- IV (stored after header, needed for decryption)

## Configuration

### Logger Configuration with Encryption

```c
blackbox_config_t config;
blackbox_get_default_config(&config);

config.root_path = "/sdcard/secure_logs";
config.file_prefix = "secure";

// *** ENABLE ENCRYPTION ***
config.encrypt = true;
memcpy(config.encryption_key, your_32_byte_key, 32);

config.file_size_limit = 256 * 1024;
config.buffer_size = 32 * 1024;
config.min_level = BLACKBOX_LOG_LEVEL_DEBUG;
config.console_output = true;   // Console shows plaintext
config.file_output = true;      // File is ENCRYPTED
```

### Example Encryption Key

⚠️ **WARNING: Never use this key in production!**

```c
static const uint8_t ENCRYPTION_KEY[32] = {
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
    0x08, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F,
    0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17,
    0x18, 0x19, 0x1A, 0x1B, 0x1C, 0x1D, 0x1E, 0x1F
};
```

## Security Best Practices

### ❌ DON'T
- Hardcode keys in source code (except for demos)
- Log or print encryption keys
- Use weak or predictable keys
- Share keys over insecure channels
- Store keys in plain text files

### ✅ DO
- Use secure key storage (encrypted NVS)
- Generate keys using cryptographic RNG
- Use unique keys per device
- Implement secure provisioning
- Rotate keys periodically
- Keep keys separate from firmware

### Key Generation Example

```python
# Generate a secure random key
import os
key = os.urandom(32)
print("Key (hex):", key.hex())
```

### Secure Key Storage (ESP32)

```c
// Store key in encrypted NVS (requires NVS encryption enabled)
nvs_handle_t handle;
nvs_open("security", NVS_READWRITE, &handle);
nvs_set_blob(handle, "log_key", key, 32);
nvs_commit(handle);
nvs_close(handle);
```

## Building and Flashing

1. Set up ESP-IDF environment:
   ```bash
   . $IDF_PATH/export.sh
   ```

2. Navigate to the example directory:
   ```bash
   cd examples/encryption_example
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
I (325) ENCRYPTED_EXAMPLE: ===========================================
I (325) ENCRYPTED_EXAMPLE:   Blackbox Logger - Encrypted Example
I (335) ENCRYPTED_EXAMPLE: ===========================================
I (385) ENCRYPTED_EXAMPLE: SD card mounted successfully!
I (395) ENCRYPTED_EXAMPLE: =========================================
W (395) ENCRYPTED_EXAMPLE: DEMO: Encryption key information
W (405) ENCRYPTED_EXAMPLE: In production, NEVER log or expose keys!
I (415) ENCRYPTED_EXAMPLE: Key (first 4 bytes): 00 01 02 03 ...
I (425) ENCRYPTED_EXAMPLE: Cipher: AES-256-CTR
I (435) ENCRYPTED_EXAMPLE: =========================================

I (445) BLACKBOX_LOG: Logger initialized: path=/sdcard/secure_logs, encrypt=1, buffer=32KB

I (455) ENCRYPTED_EXAMPLE: ========================================
I (455) ENCRYPTED_EXAMPLE:   ENCRYPTED LOGGING ENABLED!
I (465) ENCRYPTED_EXAMPLE:   - Cipher: AES-256-CTR
I (465) ENCRYPTED_EXAMPLE:   - Files are encrypted on disk
I (475) ENCRYPTED_EXAMPLE:   - Console output is plaintext
I (475) ENCRYPTED_EXAMPLE: ========================================

I (495) SECURE: [1] User: USR-12345-ABCDE | Location: 37.774912, -122.419401
D (505) SECURE: [1] Balance: $1234.56 | Access: 4521
...
```

## Decoding Encrypted Log Files

### Using the Python Decoder

```bash
# Decode with the example key (hex format)
python ../../tools/blackbox_decoder.py secure001.blackbox \
    --key 000102030405060708090A0B0C0D0E0F101112131415161718191A1B1C1D1E1F

# Save key to file and use that
echo -n -e '\x00\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0A\x0B\x0C\x0D\x0E\x0F\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1A\x1B\x1C\x1D\x1E\x1F' > key.bin
python ../../tools/blackbox_decoder.py secure001.blackbox --key-file key.bin

# Export decrypted logs to CSV
python ../../tools/blackbox_decoder.py secure001.blackbox \
    --key YOUR_KEY_HEX --format csv --output decrypted_logs.csv
```

### Decryption Requirements

- Python 3.6+
- `cryptography` library: `pip install cryptography`
- The same 32-byte key used for encryption

## File Format (Encrypted)

```
┌─────────────────────────────────────┐
│ File Header (48 bytes) - PLAINTEXT  │
│  - Magic: "ULog"                    │
│  - Version: 1                       │
│  - Flags: 0x01 (encrypted)          │
│  - Device ID                        │
│  - Timestamp                        │
├─────────────────────────────────────┤
│ IV (16 bytes) - PLAINTEXT           │
│  - Random initialization vector     │
├─────────────────────────────────────┤
│ Encrypted Data (AES-256-CTR)        │
│  - Log packets                      │
│  - Headers + payloads               │
│  - All binary data                  │
└─────────────────────────────────────┘
```

## Troubleshooting

### "Encryption key required" error
- Provide the key with `--key` or `--key-file` option
- Verify the key is exactly 32 bytes (64 hex characters)

### Decrypted data looks corrupted
- Wrong encryption key
- File was partially written (incomplete packets)
- IV mismatch (each file has unique IV)

### mbedTLS errors during build
- Ensure `sdkconfig.defaults` includes:
  ```
  CONFIG_MBEDTLS_AES_C=y
  CONFIG_MBEDTLS_CIPHER_MODE_CTR=y
  ```

### Performance impact
- AES encryption adds ~5-10% CPU overhead
- Consider reducing log frequency for battery-powered devices

## Security Audit Checklist

Before deploying encrypted logging in production:

- [ ] Keys are not hardcoded in source
- [ ] Keys are stored securely (HSM, encrypted NVS, secure element)
- [ ] Key provisioning is done securely
- [ ] Keys are unique per device (recommended)
- [ ] Key rotation procedure is documented
- [ ] Decryption tools are secured
- [ ] Log access is restricted and audited
- [ ] Encryption is verified (try decrypting with wrong key)

## Memory Usage

- Ring buffer: 32KB
- mbedTLS cipher context: ~1KB
- Stack for writer task: 4KB
- Stack for secure_data task: 4KB
- Stack for sys_monitor task: 4KB

## License

See the main project LICENSE file.
