# MP4Decrypt-WASM

A pre-compiled WebAssembly (WASM) version of mp4decrypt from the Bento4 SDK that enables high-performance decryption of CENC-encrypted MP4/M4A files directly in web browsers.


## CDN Hosting

The module is hosted on jsDelivr CDN for easy integration:

- **JavaScript Module**: `https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.js`
- **WebAssembly Binary**: `https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.wasm`

## Installation

No installation required! Import directly from the CDN:

```javascript
import createModule from 'https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.js';
```

## Usage

### Step 1: Initialize the WebAssembly Module

The module must be loaded and initialized before use. This is an asynchronous process:

```javascript
// Import the factory function from the CDN
import createModule from 'https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.js';

// Global variable to hold the initialized module instance
let mp4decryptModule;

async function initializeWasm() {
    console.log('Loading MP4Decrypt WASM Module...');
    
    // Configuration object for the module
    const moduleConfig = {
        locateFile: (path, prefix) => {
            if (path.endsWith('.wasm')) {
                return 'https://cdn.jsdelivr.net/gh/rwnk-12/mp4decrypt-wasm/mp4decrypt.wasm';
            }
            return prefix + path;
        },
        // Optional: Capture errors or logs from the C++ code
        printErr: (text) => {
            console.error('mp4decrypt.wasm:', text);
        }
    };

    try {
        mp4decryptModule = await createModule(moduleConfig);
        console.log('WASM Module Initialized Successfully.');
        // Now you can enable UI elements, etc.
    } catch (err) {
        console.error('Error initializing WASM module:', err);
    }
}

// Initialize when your app starts
initializeWasm();
```

### Step 2: Decrypt Files

Once initialized, use the module to decrypt files:

```javascript
/**
 * Decrypts an encrypted M4A file using the initialized WASM module.
 * @param {Uint8Array} encryptedData - The raw binary data of the encrypted file
 * @param {string} kid - The Key ID (32-character hex string, no hyphens)
 * @param {string} key - The Decryption Key (32-character hex string)
 * @returns {Uint8Array} The raw binary data of the decrypted file
 */
function decryptFile(encryptedData, kid, key) {
    if (!mp4decryptModule) {
        throw new Error("WASM module is not initialized.");
    }

    const encryptedFileName = 'encrypted.m4a';
    const decryptedFileName = 'decrypted.m4a';

    try {
        // 1. Write the encrypted file to the virtual filesystem
        mp4decryptModule.FS.writeFile(encryptedFileName, encryptedData);

        // 2. Construct the command-line arguments
        const commandArgs = [
            '--key',
            `${kid}:${key}`,
            encryptedFileName,
            decryptedFileName
        ];

        // 3. Call the main function
        mp4decryptModule.callMain(commandArgs);

        // 4. Check if the output file was created
        const outputExists = mp4decryptModule.FS.analyzePath(decryptedFileName).exists;
        if (!outputExists) {
            throw new Error("Decryption failed: Output file was not created.");
        }

        // 5. Read the decrypted file data from the virtual filesystem
        const decryptedData = mp4decryptModule.FS.readFile(decryptedFileName, { encoding: 'binary' });

        return decryptedData;

    } catch (error) {
        console.error("An error occurred during decryption:", error);
        throw error; // Re-throw the error to be handled by the caller
    } finally {
        // 6. Clean up the files in the virtual filesystem to free up memory
        if (mp4decryptModule.FS.analyzePath(encryptedFileName).exists) {
            mp4decryptModule.FS.unlink(encryptedFileName);
        }
        if (mp4decryptModule.FS.analyzePath(decryptedFileName).exists) {
            mp4decryptModule.FS.unlink(decryptedFileName);
        }
    }
}
```

### Complete Example

```javascript
async function downloadAndDecrypt() {
    // Ensure the WASM module is initialized
    if (!mp4decryptModule) {
        await initializeWasm();
    }

    try {
        // Your encrypted file data (Uint8Array)
        const encryptedFileBytes = /* ... your encrypted file data ... */;
        
        // Your decryption credentials (32-character hex strings)
        const decryptionKid = '/* your key ID */';
        const decryptionKey = '/* your decryption key */';

        // Decrypt the file
        const decryptedFileBytes = decryptFile(encryptedFileBytes, decryptionKid, decryptionKey);

        // Create a blob and trigger download
        const blob = new Blob([decryptedFileBytes], { type: 'audio/mp4' });
        const url = URL.createObjectURL(blob);
        
        const a = document.createElement('a');
        a.href = url;
        a.download = 'decrypted-audio.m4a';
        a.click();
        
        URL.revokeObjectURL(url);
        
    } catch (error) {
        console.error('Decryption failed:', error);
    }
}
```


## License

This project uses the Bento4 SDK. Please refer to the original Bento4 license for usage terms.

## Contributing

Issues and pull requests are welcome. 
