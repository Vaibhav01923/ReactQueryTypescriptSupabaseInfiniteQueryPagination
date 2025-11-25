# Stake Engine 3D Model Implementation Guide

This guide explains how to implement 3D models (GLTF/GLB) in a way that is compatible with the Stake Engine build environment.

## The Problem: Content Security Policy (CSP)

Stake Engine's production environment enforces a strict Content Security Policy (CSP) that blocks `blob:` URLs.

- **Standard Behavior**: `THREE.GLTFLoader` automatically creates `blob:` URLs for textures embedded within `.glb` files.
- **Result**: The browser blocks these requests, causing textures to fail to load, and often crashing the loader with errors like `Couldn't load texture blob:...`.

## The Solution: Data URI Patching

To bypass this restriction while preserving the original model's textures and mapping, we must:

1.  **Serve the GLB as a Static Asset**: Place it in `public/` to bypass Vite processing.
2.  **Patch the GLB in Memory**: Intercept the loading process to convert embedded textures into `data:` URIs (Base64), which are allowed by the CSP.

---

## Implementation Steps

### 1. Asset Placement

Do **NOT** put `.glb` files in `src/assets`.
**DO** put them in the `public/` directory.

Example:

```
/public
  /models
    chest.glb
```

### 2. The `loadModel` Function

Use the following robust function to load your models. It handles the binary parsing, texture extraction, and Data URI conversion automatically.

**`src/utils/ModelLoader.ts`** (or similar):

```typescript
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

export const loadPatchedModel = async (
  modelPath: string
): Promise<THREE.Group> => {
  return new Promise(async (resolve, reject) => {
    try {
      // 1. Fetch the GLB file as an ArrayBuffer
      const response = await fetch(modelPath);
      if (!response.ok)
        throw new Error(`Failed to fetch model: ${response.statusText}`);
      const arrayBuffer = await response.arrayBuffer();

      // 2. Parse the GLB header
      const dataView = new DataView(arrayBuffer);
      const magic = dataView.getUint32(0, true);
      const version = dataView.getUint32(4, true);
      const totalLength = dataView.getUint32(8, true);

      if (magic !== 0x46546c67) throw new Error("Invalid GLB magic number");

      // 3. Read JSON Chunk
      const jsonChunkLength = dataView.getUint32(12, true);
      const jsonChunkType = dataView.getUint32(16, true);
      if (jsonChunkType !== 0x4e4f534a)
        throw new Error("Invalid JSON chunk type");

      const jsonText = new TextDecoder().decode(
        new Uint8Array(arrayBuffer, 20, jsonChunkLength)
      );
      const json = JSON.parse(jsonText);

      // 4. Convert embedded textures to Data URIs
      // This bypasses CSP Blob blocking while preserving original assets
      const binaryChunkStart = 20 + jsonChunkLength;

      if (json.images && Array.isArray(json.images)) {
        json.images.forEach((image: any) => {
          if (image.bufferView !== undefined) {
            const bufferView = json.bufferViews[image.bufferView];
            const offset = bufferView.byteOffset || 0;
            const length = bufferView.byteLength;

            // CRITICAL: Add 8 bytes for Binary Chunk Header (Length + Type)
            const absoluteOffset = binaryChunkStart + 8 + offset;
            const imageData = new Uint8Array(
              arrayBuffer,
              absoluteOffset,
              length
            );

            // Convert to Base64
            let binary = "";
            const len = imageData.byteLength;
            for (let i = 0; i < len; i++) {
              binary += String.fromCharCode(imageData[i]);
            }
            const base64 = window.btoa(binary);

            const mimeType = image.mimeType || "image/png";
            image.uri = `data:${mimeType};base64,${base64}`;

            // Remove bufferView reference so loader uses the URI
            delete image.bufferView;
            delete image.mimeType;
          }
        });
      }

      // 5. Re-encode JSON
      const newJsonText = JSON.stringify(json);
      // Pad with spaces to 4-byte boundary
      const padding = (4 - (newJsonText.length % 4)) % 4;
      const paddedJsonText = newJsonText + " ".repeat(padding);
      const newJsonChunkLength = paddedJsonText.length;
      const newJsonBytes = new TextEncoder().encode(paddedJsonText);

      // 6. Reconstruct the GLB buffer
      const binaryChunkLength = totalLength - binaryChunkStart;

      const newTotalLength = 12 + 8 + newJsonChunkLength + binaryChunkLength;
      const newBuffer = new ArrayBuffer(newTotalLength);
      const newDataView = new DataView(newBuffer);

      // Write Header
      newDataView.setUint32(0, magic, true);
      newDataView.setUint32(4, version, true);
      newDataView.setUint32(8, newTotalLength, true);

      // Write JSON Chunk Header
      newDataView.setUint32(12, newJsonChunkLength, true);
      newDataView.setUint32(16, jsonChunkType, true);

      // Write JSON Data
      const newUint8 = new Uint8Array(newBuffer);
      newUint8.set(newJsonBytes, 20);

      // Copy Binary Chunk (if any)
      if (binaryChunkLength > 0) {
        const binaryData = new Uint8Array(
          arrayBuffer,
          binaryChunkStart,
          binaryChunkLength
        );
        newUint8.set(binaryData, 20 + newJsonChunkLength);
      }

      // 7. Load the patched GLB
      const loader = new GLTFLoader();
      loader.parse(
        newBuffer,
        "./", // path
        (gltf) => {
          resolve(gltf.scene);
        },
        (error) => {
          console.error("Error parsing patched model:", error);
          reject(error);
        }
      );
    } catch (e) {
      console.error("Error in patch/load process:", e);
      reject(e);
    }
  });
};
```

### 3. Usage

```typescript
import { loadPatchedModel } from "./utils/ModelLoader";

// ... inside your component or class
const model = await loadPatchedModel("./models/chest.glb");
scene.add(model);
```

## Troubleshooting

- **"Couldn't load texture blob:..."**: The patching logic isn't running or failed. Ensure you are using `loadPatchedModel`.
- **"Crackled" or "Glitchy" Texture**: The binary offset is likely wrong. Ensure you are adding `+ 8` to `binaryChunkStart` when reading image data (as shown in the code above).
- **Model not found (404)**: Ensure the file is in `public/` and you are referencing it with a relative path like `./filename.glb`.
