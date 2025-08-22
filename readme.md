# mp4decrypt-wasm

A browser-compatible WASM version of mp4decrypt for in-browser decryption of MP4 files.

## Quick Start

### Installation

Use the CDN links directly in your browser:

- **JS Module:** `https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.js`
- **WASM Binary:** `https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.wasm`

> **Note:** Your page must run as an ES module (use `<script type="module">` in HTML).

### Basic Usage

```javascript
import createModule from 'https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.js';

let decryptionModule;

async function initMp4decrypt() {
  const wasmUrl = 'https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.wasm';
  const wasmBinary = await fetch(wasmUrl).then(response => {
    if (!response.ok) throw new Error(`Failed to fetch WASM module: ${response.statusText}`);
    return response.arrayBuffer();
  });
  
  let capturedStdErr = '';
  const moduleArgs = { 
    wasmBinary,
    printErr: (text) => { capturedStdErr += text + '\n'; }
  };
  
  decryptionModule = await createModule(moduleArgs);
  
  // Add helper function to get and clear stderr
  decryptionModule.capturedStdErr = () => { 
    const err = capturedStdErr; 
    capturedStdErr = ''; 
    return err; 
  };
}

// Initialize before using
await initMp4decrypt();
```

## Decryption Example

```javascript
function decryptFile(encryptedBuffer, decryptionKey) {
  const encryptedFileName = 'encrypted.m4a';
  const decryptedFileName = 'decrypted.m4a';
  
  // Write encrypted data to virtual filesystem
  decryptionModule.FS.writeFile(encryptedFileName, new Uint8Array(encryptedBuffer));
  
  // Run decryption command
  const commandArgs = ['--key', `1:${decryptionKey}`, encryptedFileName, decryptedFileName];
  decryptionModule.callMain(commandArgs);
  
  // Check if decryption succeeded
  if (!decryptionModule.FS.analyzePath(decryptedFileName).exists) {
    throw new Error(`Decryption failed. Check key and input file.`);
  }
  
  // Read decrypted data
  const decryptedBuffer = decryptionModule.FS.readFile(decryptedFileName).buffer;
  
  // Clean up temporary files
  decryptionModule.FS.unlink(encryptedFileName);
  decryptionModule.FS.unlink(decryptedFileName);
  
  return decryptedBuffer;
}

// Usage example
try {
  const decryptedData = decryptFile(myEncryptedBuffer, 'YOUR_HEX_KEY_HERE');
  console.log('Decryption successful!', decryptedData);
} catch (error) {
  console.error('Decryption failed:', error.message);
  console.error('stderr:', decryptionModule.capturedStdErr());
}
```

## API Reference

| Method | Description |
|--------|-------------|
| `callMain(args: string[])` | Run mp4decrypt with CLI arguments array |
| `FS.writeFile(filename, Uint8Array)` | Write file to in-memory filesystem |
| `FS.readFile(filename)` | Read file as Uint8Array from filesystem |
| `FS.unlink(filename)` | Remove file from filesystem (important for cleanup) |
| `FS.analyzePath(filename)` | Returns object with `{ exists: boolean, ... }` |
| `capturedStdErr()` | Get and clear captured stderr output (custom helper) |

## Command Arguments

### Key Format

```bash
# Key format for --key argument
--key <track_id>:<key>

# Example command line equivalent:
mp4decrypt --key 1:YOUR_32_CHAR_HEX_KEY input.m4a output.m4a
```

### Usage Examples

```javascript
// Single key example:
decryptionModule.callMain([
  '--key', '1:abcdef1234567890abcdef1234567890',
  'input.m4a', 'output.m4a'
]);

// Multiple keys example:
decryptionModule.callMain([
  '--key', '1:key1_hex_here',
  '--key', '2:key2_hex_here',
  'input.m4a', 'output.m4a'
]);
```

> **Warning:** Keys should be 32-character hexadecimal strings (128-bit keys). Track ID typically starts from 1.

## Error Handling

> **Important:** Do NOT rely on try-catch around `callMain()`. Always check output file existence for success/failure.

```javascript
// Correct error handling approach:
decryptionModule.callMain(commandArgs);

if (!decryptionModule.FS.analyzePath(outputFile).exists) {
  // Decryption failed - check stderr for details
  const errorOutput = decryptionModule.capturedStdErr();
  throw new Error(`Decryption failed: ${errorOutput}`);
}
// Success - file exists
```

### Common Failure Reasons

- Wrong decryption key
- Incorrect track ID
- Corrupted or incomplete input file
- Input file is not encrypted or uses different encryption

## Browser Requirements & Setup

### HTML Setup

```html
<!-- Essential: Use type="module" for ES module support -->
<script type="module">
  import createModule from 'https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.js';
  // Your code here
</script>

<!-- OR external module file -->
<script type="module" src="your-script.js"></script>
```

### Requirements

- **ES Modules:** Must use `<script type="module">` or bundler
- **HTTPS:** Required for CDN access and WASM loading (not file:///)
- **Modern Browser:** WebAssembly support (Chrome 57+, Firefox 52+, Safari 11+)
- **Memory:** Sufficient RAM for file processing (encrypted file size + overhead)

### CSP Policy

If using Content Security Policy, allow:
- `script-src 'self' https://cdn.jsdelivr.net`
- `connect-src 'self' https://cdn.jsdelivr.net`
- `unsafe-eval` may be required for WASM in some browsers

> **Note:** First-time visitors will experience slower performance as the WASM module (several MB) downloads and compiles. Consider showing a loading indicator.

## Filesystem Best Practices

- **All files are in-memory** - no persistence after page reload
- **Always cleanup:** Use `FS.unlink()` for input and output files
- **Unique filenames:** Use timestamps or UUIDs for concurrent operations
- **Memory management:** Large files consume browser memory until cleaned up

```javascript
// Good practice for concurrent operations:
function decryptWithUniqueNames(encryptedBuffer, key) {
  const timestamp = Date.now();
  const inputFile = `encrypted_${timestamp}.m4a`;
  const outputFile = `decrypted_${timestamp}.m4a`;
  
  try {
    decryptionModule.FS.writeFile(inputFile, new Uint8Array(encryptedBuffer));
    decryptionModule.callMain(['--key', `1:${key}`, inputFile, outputFile]);
    
    if (!decryptionModule.FS.analyzePath(outputFile).exists) {
      throw new Error('Decryption failed');
    }
    
    return decryptionModule.FS.readFile(outputFile).buffer;
  } finally {
    // Always cleanup, even if error occurs
    try { decryptionModule.FS.unlink(inputFile); } catch {}
    try { decryptionModule.FS.unlink(outputFile); } catch {}
  }
}
```

## License

This project is a WebAssembly port of the original mp4decrypt tool. Please refer to the original project's licensing terms.
