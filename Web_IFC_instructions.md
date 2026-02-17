# Web-IFC: Complete Implementation Guide for AI Agents

> **Library:** `web-ifc` (npm)
> **Version at time of writing:** 0.0.76
> **Repository:** [ThatOpen/engine_web-ifc](https://github.com/ThatOpen/engine_web-ifc)
> **License:** MPL-2.0
> **Purpose:** Parse, read, write, and generate geometry from IFC (Industry Foundation Classes) building model files entirely in the browser or Node.js via WebAssembly.

---

## Table of Contents

1. [What is web-ifc?](#1-what-is-web-ifc)
2. [Installation](#2-installation)
3. [WASM File Setup (Critical)](#3-wasm-file-setup-critical)
4. [Initialization](#4-initialization)
5. [Opening and Loading IFC Files](#5-opening-and-loading-ifc-files)
6. [Three.js Integration (Complete)](#6-threejs-integration-complete)
7. [React + Three.js Integration](#7-react--threejs-integration)
8. [Property Reading](#8-property-reading)
9. [Spatial Structure (Building Hierarchy)](#9-spatial-structure-building-hierarchy)
10. [Property Sets and Materials](#10-property-sets-and-materials)
11. [Writing and Modifying IFC Data](#11-writing-and-modifying-ifc-data)
12. [Creating New IFC Models from Scratch](#12-creating-new-ifc-models-from-scratch)
13. [Saving / Exporting IFC Files](#13-saving--exporting-ifc-files)
14. [Picking and Selection](#14-picking-and-selection)
15. [Filtering by IFC Type](#15-filtering-by-ifc-type)
16. [Coordination Matrix and Transformations](#16-coordination-matrix-and-transformations)
17. [GUID / ExpressID Mapping](#17-guid--expressid-mapping)
18. [Streaming API (Memory-Efficient Large Files)](#18-streaming-api-memory-efficient-large-files)
19. [Multi-Threading](#19-multi-threading)
20. [Firebase Integration Patterns](#20-firebase-integration-patterns)
21. [LoaderSettings Reference](#21-loadersettings-reference)
22. [Complete Type Reference](#22-complete-type-reference)
23. [Complete API Method Reference](#23-complete-api-method-reference)
24. [IFC Type Constants](#24-ifc-type-constants)
25. [Logging and Debugging](#25-logging-and-debugging)
26. [Performance Best Practices](#26-performance-best-practices)
27. [Common Pitfalls](#27-common-pitfalls)
28. [Updating to Future Versions](#28-updating-to-future-versions)

---

## 1. What is web-ifc?

`web-ifc` is a low-level IFC engine compiled to WebAssembly. It can:

- **Parse** IFC files (IFC2X3, IFC4, IFC4x1, IFC4x2, IFC4x3) into structured data
- **Generate geometry** (meshes with vertices, normals, and indices) from IFC geometric representations
- **Read/write properties** on individual IFC entities
- **Navigate the spatial structure** (Site → Building → Storey → Space → Elements)
- **Create new IFC models** programmatically
- **Save/export** models back to valid IFC STEP files
- **Stream meshes** for memory-efficient processing of large files

It does NOT include a viewer — you pair it with Three.js (or any other 3D engine) for rendering.

---

## 2. Installation

```bash
npm install web-ifc
```

This installs:
- `web-ifc-api.js` — Browser ESM bundle
- `web-ifc-api-node.js` — Node.js CommonJS bundle
- `web-ifc.wasm` — Single-threaded WebAssembly binary
- `web-ifc-mt.wasm` — Multi-threaded WebAssembly binary
- `web-ifc-mt.worker.js` — Web Worker for multi-threading
- TypeScript declaration files (`.d.ts`)

For a Three.js React app you also need:
```bash
npm install three @types/three @react-three/fiber @react-three/drei
```

---

## 3. WASM File Setup (Critical)

The `.wasm` files must be served as static assets accessible by the browser at runtime. web-ifc needs to fetch these files when `Init()` is called.

### Vite (recommended for React)

```js
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  optimizeDeps: {
    exclude: ['web-ifc'], // Prevent Vite from pre-bundling WASM
  },
});
```

Then copy the WASM files to your `public/` folder:

```bash
cp node_modules/web-ifc/web-ifc.wasm public/
cp node_modules/web-ifc/web-ifc-mt.wasm public/
cp node_modules/web-ifc/web-ifc-mt.worker.js public/
```

Or automate with a script in `package.json`:
```json
{
  "scripts": {
    "copy-wasm": "cp node_modules/web-ifc/*.wasm public/ && cp node_modules/web-ifc/*.worker.js public/",
    "postinstall": "npm run copy-wasm"
  }
}
```

### Create React App (Webpack)

Copy WASM files to `public/` and set the path:
```typescript
ifcApi.SetWasmPath("./");
```

### Next.js

Place WASM files in the `public/` directory and set:
```typescript
ifcApi.SetWasmPath("/");
```

### Custom path

You can place WASM files anywhere and tell web-ifc where to find them:
```typescript
// Relative path (relative to the page URL)
ifcApi.SetWasmPath("./wasm/");

// Absolute path
ifcApi.SetWasmPath("/static/wasm/", true);

// CDN or remote URL via custom locator
await ifcApi.Init((path) => "https://cdn.example.com/wasm/" + path);
```

---

## 4. Initialization

```typescript
import { IfcAPI } from "web-ifc";

const ifcApi = new IfcAPI();

// Option A: Default WASM path (looks in same directory as the page)
await ifcApi.Init();

// Option B: Custom WASM path
ifcApi.SetWasmPath("/wasm/");
await ifcApi.Init();

// Option C: Custom file locator function
await ifcApi.Init((fileName) => `/static/wasm/${fileName}.wasm`);

// Option D: Force single-threaded mode (useful for debugging or compatibility)
await ifcApi.Init(undefined, true);
```

**Important:** `Init()` is async and MUST complete before calling any other method. Call it once per `IfcAPI` instance.

---

## 5. Opening and Loading IFC Files

### From a File Input (browser)

```typescript
async function loadIFC(file: File): Promise<number> {
  const data = new Uint8Array(await file.arrayBuffer());
  const modelID = ifcApi.OpenModel(data);
  return modelID;
}
```

### From a URL (fetch)

```typescript
async function loadIFCFromURL(url: string): Promise<number> {
  const response = await fetch(url);
  const buffer = await response.arrayBuffer();
  const data = new Uint8Array(buffer);
  const modelID = ifcApi.OpenModel(data);
  return modelID;
}
```

### From Firebase Storage

```typescript
import { getStorage, ref, getBytes } from "firebase/storage";

async function loadIFCFromFirebase(filePath: string): Promise<number> {
  const storage = getStorage();
  const fileRef = ref(storage, filePath);
  const buffer = await getBytes(fileRef);
  const data = new Uint8Array(buffer);
  const modelID = ifcApi.OpenModel(data);
  return modelID;
}
```

### With Custom Settings

```typescript
const modelID = ifcApi.OpenModel(data, {
  COORDINATE_TO_ORIGIN: true,   // Center the model at world origin
  CIRCLE_SEGMENTS: 16,          // Smoother circles (default 12)
  MEMORY_LIMIT: 2147483648,     // 2GB max WASM memory
});
```

### Streaming Large Files (callback-based)

```typescript
const modelID = ifcApi.OpenModelFromCallback((offset, size) => {
  // Return a Uint8Array chunk of `size` bytes starting at `offset`
  return fileData.subarray(offset, offset + size);
});
```

### Closing Models (free memory)

```typescript
ifcApi.CloseModel(modelID);   // Close a specific model
ifcApi.CloseAllModels();       // Close all models
ifcApi.Dispose();              // Fully shut down the WASM module
```

**Always close models when done to prevent memory leaks.** The WASM heap does not get garbage-collected automatically.

---

## 6. Three.js Integration (Complete)

This is the core geometry-to-Three.js conversion. The vertex format from web-ifc is interleaved: 6 floats per vertex = `[posX, posY, posZ, normX, normY, normZ]`.

### Full IfcThreeAdapter Class

```typescript
import * as THREE from "three";
import { IfcAPI, FlatMesh, PlacedGeometry, Color } from "web-ifc";
import { mergeGeometries } from "three/addons/utils/BufferGeometryUtils.js";

/**
 * Converts web-ifc geometry into Three.js meshes.
 *
 * VERTEX FORMAT: web-ifc returns 6 floats per vertex:
 *   [positionX, positionY, positionZ, normalX, normalY, normalZ]
 *
 * Each PlacedGeometry has:
 *   - geometryExpressID: reference to the shared geometry
 *   - color: { x: R, y: G, z: B, w: Alpha } (0-1 range)
 *   - flatTransformation: number[16] — column-major 4x4 transform matrix
 */
export class IfcThreeAdapter {
  private ifcApi: IfcAPI;
  private materialCache: Map<string, THREE.MeshPhongMaterial> = new Map();

  constructor(ifcApi: IfcAPI) {
    this.ifcApi = ifcApi;
  }

  /**
   * Loads all geometry from a model into the scene.
   * Uses streaming to keep memory usage low.
   * Returns a map of expressID → THREE.Mesh for picking.
   */
  loadAllGeometry(modelID: number): {
    opaqueMesh: THREE.Mesh | null;
    transparentMesh: THREE.Mesh | null;
    expressIdToGeometryIndex: Map<number, number>;
  } {
    const opaqueGeometries: THREE.BufferGeometry[] = [];
    const transparentGeometries: THREE.BufferGeometry[] = [];
    const expressIdToGeometryIndex = new Map<number, number>();

    this.ifcApi.StreamAllMeshes(modelID, (mesh: FlatMesh) => {
      const placedGeometries = mesh.geometries;

      for (let i = 0; i < placedGeometries.size(); i++) {
        const placedGeometry = placedGeometries.get(i);
        const threeGeometry = this.placedGeometryToBuffer(
          modelID,
          placedGeometry
        );

        if (placedGeometry.color.w < 1) {
          expressIdToGeometryIndex.set(
            mesh.expressID,
            transparentGeometries.length
          );
          transparentGeometries.push(threeGeometry);
        } else {
          expressIdToGeometryIndex.set(
            mesh.expressID,
            opaqueGeometries.length
          );
          opaqueGeometries.push(threeGeometry);
        }
      }
    });

    let opaqueMesh: THREE.Mesh | null = null;
    if (opaqueGeometries.length > 0) {
      const merged = mergeGeometries(opaqueGeometries);
      const material = new THREE.MeshPhongMaterial({
        side: THREE.DoubleSide,
        vertexColors: true,
      });
      opaqueMesh = new THREE.Mesh(merged, material);
    }

    let transparentMesh: THREE.Mesh | null = null;
    if (transparentGeometries.length > 0) {
      const merged = mergeGeometries(transparentGeometries);
      const material = new THREE.MeshPhongMaterial({
        side: THREE.DoubleSide,
        vertexColors: true,
        transparent: true,
        opacity: 0.5,
      });
      transparentMesh = new THREE.Mesh(merged, material);
    }

    return { opaqueMesh, transparentMesh, expressIdToGeometryIndex };
  }

  /**
   * Loads a single FlatMesh (single element) into a Three.js Mesh.
   */
  loadSingleElement(modelID: number, expressID: number): THREE.Mesh | null {
    const flatMesh = this.ifcApi.GetFlatMesh(modelID, expressID);
    const placedGeometries = flatMesh.geometries;
    const geometries: THREE.BufferGeometry[] = [];

    for (let i = 0; i < placedGeometries.size(); i++) {
      geometries.push(
        this.placedGeometryToBuffer(modelID, placedGeometries.get(i))
      );
    }

    if (geometries.length === 0) return null;

    const merged =
      geometries.length === 1
        ? geometries[0]
        : mergeGeometries(geometries);

    const color = placedGeometries.get(0).color;
    const material = this.getMaterial(color);
    return new THREE.Mesh(merged, material);
  }

  /**
   * Converts a PlacedGeometry into a Three.js BufferGeometry
   * with the transformation matrix already applied.
   */
  private placedGeometryToBuffer(
    modelID: number,
    placedGeometry: PlacedGeometry
  ): THREE.BufferGeometry {
    const ifcGeometry = this.ifcApi.GetGeometry(
      modelID,
      placedGeometry.geometryExpressID
    );

    const verts = this.ifcApi.GetVertexArray(
      ifcGeometry.GetVertexData(),
      ifcGeometry.GetVertexDataSize()
    );

    const indices = this.ifcApi.GetIndexArray(
      ifcGeometry.GetIndexData(),
      ifcGeometry.GetIndexDataSize()
    );

    const geometry = this.ifcVertsToBufferGeometry(
      placedGeometry.color,
      verts,
      indices
    );

    // Apply the 4x4 placement transform matrix
    const matrix = new THREE.Matrix4();
    matrix.fromArray(placedGeometry.flatTransformation);
    geometry.applyMatrix4(matrix);

    // CRITICAL: delete the WASM geometry to free memory
    // @ts-ignore — delete() is an Emscripten binding method
    ifcGeometry.delete();

    return geometry;
  }

  /**
   * Converts raw interleaved vertex data [pos.xyz, norm.xyz]
   * into a Three.js BufferGeometry with separate position, normal, and color attributes.
   */
  private ifcVertsToBufferGeometry(
    color: Color,
    vertexData: Float32Array,
    indexData: Uint32Array
  ): THREE.BufferGeometry {
    const geometry = new THREE.BufferGeometry();

    const vertexCount = vertexData.length / 6;
    const positions = new Float32Array(vertexCount * 3);
    const normals = new Float32Array(vertexCount * 3);
    const colors = new Float32Array(vertexCount * 3);

    for (let i = 0; i < vertexData.length; i += 6) {
      const outIdx = (i / 6) * 3;
      // Position
      positions[outIdx + 0] = vertexData[i + 0];
      positions[outIdx + 1] = vertexData[i + 1];
      positions[outIdx + 2] = vertexData[i + 2];
      // Normal
      normals[outIdx + 0] = vertexData[i + 3];
      normals[outIdx + 1] = vertexData[i + 4];
      normals[outIdx + 2] = vertexData[i + 5];
      // Vertex Color (same for all vertices of this geometry)
      colors[outIdx + 0] = color.x;
      colors[outIdx + 1] = color.y;
      colors[outIdx + 2] = color.z;
    }

    geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
    geometry.setAttribute("color", new THREE.BufferAttribute(colors, 3));
    geometry.setIndex(new THREE.BufferAttribute(indexData, 1));

    return geometry;
  }

  private getMaterial(color: Color): THREE.MeshPhongMaterial {
    const key = `${color.x}_${color.y}_${color.z}_${color.w}`;
    if (this.materialCache.has(key)) return this.materialCache.get(key)!;

    const material = new THREE.MeshPhongMaterial({
      color: new THREE.Color(color.x, color.y, color.z),
      side: THREE.DoubleSide,
    });
    if (color.w < 1) {
      material.transparent = true;
      material.opacity = color.w;
    }
    this.materialCache.set(key, material);
    return material;
  }
}
```

---

## 7. React + Three.js Integration

### Recommended Architecture

```
src/
├── hooks/
│   └── useIfcLoader.ts       # IfcAPI lifecycle + file loading
├── components/
│   ├── IfcViewer.tsx          # R3F Canvas wrapper
│   ├── IfcModel.tsx           # Renders the IFC mesh in the scene
│   └── IfcFileInput.tsx       # File upload UI
├── utils/
│   └── ifcThreeAdapter.ts    # The IfcThreeAdapter class from Section 6
└── App.tsx
```

### useIfcLoader Hook

```typescript
// hooks/useIfcLoader.ts
import { useEffect, useRef, useState, useCallback } from "react";
import { IfcAPI } from "web-ifc";

export function useIfcLoader() {
  const ifcApiRef = useRef<IfcAPI | null>(null);
  const [isReady, setIsReady] = useState(false);
  const [modelID, setModelID] = useState<number | null>(null);

  useEffect(() => {
    const api = new IfcAPI();
    ifcApiRef.current = api;

    // Set the path to where your WASM files are served
    api.SetWasmPath("/");

    api.Init().then(() => {
      setIsReady(true);
    });

    return () => {
      api.Dispose();
      ifcApiRef.current = null;
    };
  }, []);

  const loadFile = useCallback(async (file: File) => {
    if (!ifcApiRef.current || !isReady) return null;

    // Close any previously open model
    if (modelID !== null) {
      ifcApiRef.current.CloseModel(modelID);
    }

    const data = new Uint8Array(await file.arrayBuffer());
    const newModelID = ifcApiRef.current.OpenModel(data, {
      COORDINATE_TO_ORIGIN: true,
    });
    setModelID(newModelID);
    return newModelID;
  }, [isReady, modelID]);

  const loadFromURL = useCallback(async (url: string) => {
    if (!ifcApiRef.current || !isReady) return null;

    if (modelID !== null) {
      ifcApiRef.current.CloseModel(modelID);
    }

    const response = await fetch(url);
    const data = new Uint8Array(await response.arrayBuffer());
    const newModelID = ifcApiRef.current.OpenModel(data, {
      COORDINATE_TO_ORIGIN: true,
    });
    setModelID(newModelID);
    return newModelID;
  }, [isReady, modelID]);

  return {
    ifcApi: ifcApiRef.current,
    isReady,
    modelID,
    loadFile,
    loadFromURL,
  };
}
```

### IfcModel Component (R3F)

```tsx
// components/IfcModel.tsx
import { useEffect, useState } from "react";
import * as THREE from "three";
import { IfcAPI } from "web-ifc";
import { IfcThreeAdapter } from "../utils/ifcThreeAdapter";

interface IfcModelProps {
  ifcApi: IfcAPI;
  modelID: number;
}

export function IfcModel({ ifcApi, modelID }: IfcModelProps) {
  const [meshes, setMeshes] = useState<{
    opaque: THREE.Mesh | null;
    transparent: THREE.Mesh | null;
  }>({ opaque: null, transparent: null });

  useEffect(() => {
    const adapter = new IfcThreeAdapter(ifcApi);
    const result = adapter.loadAllGeometry(modelID);
    setMeshes({
      opaque: result.opaqueMesh,
      transparent: result.transparentMesh,
    });

    return () => {
      // Dispose Three.js geometries on unmount
      result.opaqueMesh?.geometry.dispose();
      result.transparentMesh?.geometry.dispose();
    };
  }, [ifcApi, modelID]);

  return (
    <group>
      {meshes.opaque && <primitive object={meshes.opaque} />}
      {meshes.transparent && <primitive object={meshes.transparent} />}
    </group>
  );
}
```

### IfcViewer Component

```tsx
// components/IfcViewer.tsx
import { Canvas } from "@react-three/fiber";
import { OrbitControls, Grid } from "@react-three/drei";
import { IfcModel } from "./IfcModel";
import { IfcAPI } from "web-ifc";

interface IfcViewerProps {
  ifcApi: IfcAPI | null;
  modelID: number | null;
}

export function IfcViewer({ ifcApi, modelID }: IfcViewerProps) {
  return (
    <Canvas camera={{ position: [15, 15, 15], fov: 50 }}>
      <ambientLight intensity={0.5} />
      <directionalLight position={[10, 20, 10]} intensity={1} />
      <OrbitControls />
      <Grid infiniteGrid fadeDistance={50} />
      {ifcApi && modelID !== null && (
        <IfcModel ifcApi={ifcApi} modelID={modelID} />
      )}
    </Canvas>
  );
}
```

### App Component

```tsx
// App.tsx
import { useIfcLoader } from "./hooks/useIfcLoader";
import { IfcViewer } from "./components/IfcViewer";

export default function App() {
  const { ifcApi, isReady, modelID, loadFile } = useIfcLoader();

  const handleFileChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) await loadFile(file);
  };

  return (
    <div style={{ width: "100vw", height: "100vh" }}>
      <input
        type="file"
        accept=".ifc"
        onChange={handleFileChange}
        disabled={!isReady}
        style={{ position: "absolute", zIndex: 1, top: 10, left: 10 }}
      />
      <IfcViewer ifcApi={ifcApi} modelID={modelID} />
    </div>
  );
}
```

---

## 8. Property Reading

### Get All Properties of an Element

```typescript
// Get the full line data for an element
// `flatten: true` resolves all nested references recursively
const element = ifcApi.GetLine(modelID, expressID, true);

console.log(element.type);           // e.g. 3856911033 (IFCWALL)
console.log(element.GlobalId?.value); // e.g. "2O2Fr$t4X7Zf8NOew3FNr2"
console.log(element.Name?.value);     // e.g. "Basic Wall:Generic - 200mm"
```

### Get Properties Using the Properties Helper

```typescript
// Get all properties of an element (most convenient method)
const props = await ifcApi.properties.getItemProperties(modelID, expressID, true);

// Get property sets
const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);

// Get type properties
const typeProps = await ifcApi.properties.getTypeProperties(modelID, expressID, true);

// Get material properties
const materials = await ifcApi.properties.getMaterialsProperties(modelID, expressID, true);
```

### Get All Elements of a Type

```typescript
import { IFCWALL, IFCSLAB, IFCCOLUMN, IFCDOOR, IFCWINDOW } from "web-ifc";

// Get all wall IDs
const wallIDs = ifcApi.GetLineIDsWithType(modelID, IFCWALL, true);

// Iterate and read properties
for (let i = 0; i < wallIDs.size(); i++) {
  const wallID = wallIDs.get(i);
  const wall = ifcApi.GetLine(modelID, wallID, true);
  console.log(`Wall: ${wall.Name?.value} (ID: ${wallID})`);
}
```

### Get Raw Line Data (lower-level)

```typescript
const rawData = ifcApi.GetRawLineData(modelID, expressID);
console.log(rawData.ID);        // Express ID number
console.log(rawData.type);      // IFC type code
console.log(rawData.arguments); // Array of raw parsed arguments
```

---

## 9. Spatial Structure (Building Hierarchy)

The spatial structure represents the hierarchical organization of a building:
`IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey → IfcSpace → Elements`

```typescript
// Get the full spatial tree
const structure = await ifcApi.properties.getSpatialStructure(modelID, false);

// Structure shape:
// {
//   expressID: number,
//   type: string,       // e.g. "IFCPROJECT"
//   children: [
//     {
//       expressID: number,
//       type: "IFCSITE",
//       children: [
//         {
//           expressID: number,
//           type: "IFCBUILDING",
//           children: [
//             {
//               expressID: number,
//               type: "IFCBUILDINGSTOREY",
//               children: [ /* elements */ ]
//             }
//           ]
//         }
//       ]
//     }
//   ]
// }

// With properties included (slower, more data)
const structureWithProps = await ifcApi.properties.getSpatialStructure(modelID, true);
```

### Recursive Tree Walk Example

```typescript
function walkSpatialTree(node: any, depth = 0) {
  const indent = "  ".repeat(depth);
  console.log(`${indent}${node.type} [${node.expressID}]`);

  if (node.children) {
    for (const child of node.children) {
      walkSpatialTree(child, depth + 1);
    }
  }
}

const tree = await ifcApi.properties.getSpatialStructure(modelID, false);
walkSpatialTree(tree);
```

---

## 10. Property Sets and Materials

### Reading Property Sets

```typescript
// Get all property sets for an element
const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);

// Each pset is an IfcPropertySet with HasProperties containing IfcPropertySingleValue entries
// Example output:
// [{
//   expressID: 456,
//   type: "IFCPROPERTYSET",
//   Name: { value: "Pset_WallCommon" },
//   HasProperties: [
//     { Name: { value: "IsExternal" }, NominalValue: { value: true } },
//     { Name: { value: "ThermalTransmittance" }, NominalValue: { value: 0.24 } }
//   ]
// }]
```

### Reading Material Information

```typescript
const materials = await ifcApi.properties.getMaterialsProperties(
  modelID,
  expressID,
  true,  // recursive
  true   // include type materials
);

// Returns material associations like IfcMaterial, IfcMaterialLayerSet, etc.
```

### Adding a Property Set to an Element

```typescript
import {
  IFCPROPERTYSET, IFCPROPERTYSINGLEVALUE, IFCRELDEFINESBYPROPERTIES,
  IFCTEXT, IFCREAL, IFCGLOBALLYUNIQUEID, IFCOWNERHISTORY
} from "web-ifc";

// 1. Create property values
const prop1 = ifcApi.CreateIfcEntity(modelID, IFCPROPERTYSINGLEVALUE,
  ifcApi.CreateIfcType(modelID, IFCTEXT, "MyProperty"),
  null,  // description
  ifcApi.CreateIfcType(modelID, IFCREAL, 42.0),
  null   // unit
);
ifcApi.WriteLine(modelID, prop1);

// 2. Create the property set
const pset = ifcApi.CreateIfcEntity(modelID, IFCPROPERTYSET,
  ifcApi.CreateIfcType(modelID, IFCGLOBALLYUNIQUEID, ifcApi.CreateIFCGloballyUniqueId(modelID)),
  null,  // OwnerHistory
  ifcApi.CreateIfcType(modelID, IFCTEXT, "Custom_Pset"),
  null,  // Description
  [prop1]
);
ifcApi.WriteLine(modelID, pset);

// 3. Associate the pset with an element via relationship
await ifcApi.properties.setPropertySets(modelID, elementExpressID, pset.expressID);
```

---

## 11. Writing and Modifying IFC Data

### Modify an Existing Property

```typescript
// 1. Get the element
const wall = ifcApi.GetLine(modelID, wallExpressID, true);

// 2. Modify a property
wall.Name.value = "Renamed Wall";

// 3. Write it back
ifcApi.WriteLine(modelID, wall);
```

### Delete an Element

```typescript
ifcApi.DeleteLine(modelID, expressID);
```

### Create a New Entity

```typescript
import { IFCWALL, IFCTEXT, IFCGLOBALLYUNIQUEID } from "web-ifc";

const newWall = ifcApi.CreateIfcEntity(
  modelID,
  IFCWALL,
  ifcApi.CreateIfcType(modelID, IFCGLOBALLYUNIQUEID,
    ifcApi.CreateIFCGloballyUniqueId(modelID)),  // GlobalId
  null,                                            // OwnerHistory
  ifcApi.CreateIfcType(modelID, IFCTEXT, "New Wall"), // Name
  null,                                            // Description
  null,                                            // ObjectType
  null,                                            // ObjectPlacement
  null,                                            // Representation
  null                                             // Tag
);

ifcApi.WriteLine(modelID, newWall);
```

---

## 12. Creating New IFC Models from Scratch

```typescript
import { IfcAPI, IFCPROJECT, IFCSITE, IFCTEXT, IFCGLOBALLYUNIQUEID } from "web-ifc";

const ifcApi = new IfcAPI();
await ifcApi.Init();

// Create a new empty model
const modelID = ifcApi.CreateModel({
  schema: "IFC4",                              // Required: "IFC2X3" | "IFC4" | "IFC4X3"
  name: "My Project",                          // Optional
  description: ["Description of my project"],  // Optional
  authors: ["Author Name"],                    // Optional
  organizations: ["Company Name"],             // Optional
  authorization: "None",                       // Optional
});

// Create the project entity
const project = ifcApi.CreateIfcEntity(modelID, IFCPROJECT,
  ifcApi.CreateIfcType(modelID, IFCGLOBALLYUNIQUEID,
    ifcApi.CreateIFCGloballyUniqueId(modelID)),
  null,
  ifcApi.CreateIfcType(modelID, IFCTEXT, "My Project"),
  null, null, null, null, null, null
);
ifcApi.WriteLine(modelID, project);

// Continue creating Site, Building, Storey, and elements...
```

---

## 13. Saving / Exporting IFC Files

### Save to Uint8Array

```typescript
const data: Uint8Array = ifcApi.SaveModel(modelID);

// Download in browser
const blob = new Blob([data], { type: "application/octet-stream" });
const url = URL.createObjectURL(blob);
const a = document.createElement("a");
a.href = url;
a.download = "model.ifc";
a.click();
URL.revokeObjectURL(url);
```

### Save to Firebase Storage

```typescript
import { getStorage, ref, uploadBytes } from "firebase/storage";

async function saveToFirebase(modelID: number, filePath: string) {
  const data = ifcApi.SaveModel(modelID);
  const storage = getStorage();
  const fileRef = ref(storage, filePath);
  await uploadBytes(fileRef, data);
}
```

### Save with Streaming Callback (large files)

```typescript
const chunks: Uint8Array[] = [];
ifcApi.SaveModelToCallback(modelID, (chunk: Uint8Array) => {
  chunks.push(chunk);
});
const fullData = new Uint8Array(
  chunks.reduce((acc, c) => acc + c.length, 0)
);
let offset = 0;
for (const chunk of chunks) {
  fullData.set(chunk, offset);
  offset += chunk.length;
}
```

---

## 14. Picking and Selection

To enable clicking on IFC elements in Three.js and identifying which IFC element was selected:

### Strategy: Separate Meshes Per Element

For picking, instead of merging all geometry into one mesh, keep individual meshes and store the expressID:

```typescript
function loadWithPicking(
  ifcApi: IfcAPI,
  modelID: number,
  scene: THREE.Scene
): Map<THREE.Mesh, number> {
  const meshToExpressId = new Map<THREE.Mesh, number>();

  ifcApi.StreamAllMeshes(modelID, (flatMesh: FlatMesh) => {
    const placedGeometries = flatMesh.geometries;

    for (let i = 0; i < placedGeometries.size(); i++) {
      const pg = placedGeometries.get(i);
      const ifcGeom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);

      const verts = ifcApi.GetVertexArray(
        ifcGeom.GetVertexData(),
        ifcGeom.GetVertexDataSize()
      );
      const indices = ifcApi.GetIndexArray(
        ifcGeom.GetIndexData(),
        ifcGeom.GetIndexDataSize()
      );

      const geometry = new THREE.BufferGeometry();
      const positions = new Float32Array(verts.length / 2);
      const normals = new Float32Array(verts.length / 2);

      for (let j = 0; j < verts.length; j += 6) {
        const idx = (j / 6) * 3;
        positions[idx] = verts[j];
        positions[idx + 1] = verts[j + 1];
        positions[idx + 2] = verts[j + 2];
        normals[idx] = verts[j + 3];
        normals[idx + 1] = verts[j + 4];
        normals[idx + 2] = verts[j + 5];
      }

      geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
      geometry.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
      geometry.setIndex(new THREE.BufferAttribute(indices, 1));

      const material = new THREE.MeshPhongMaterial({
        color: new THREE.Color(pg.color.x, pg.color.y, pg.color.z),
        side: THREE.DoubleSide,
        transparent: pg.color.w < 1,
        opacity: pg.color.w,
      });

      const mesh = new THREE.Mesh(geometry, material);
      const matrix = new THREE.Matrix4().fromArray(pg.flatTransformation);
      mesh.applyMatrix4(matrix);

      scene.add(mesh);
      meshToExpressId.set(mesh, flatMesh.expressID);

      // @ts-ignore
      ifcGeom.delete();
    }
  });

  return meshToExpressId;
}
```

### Raycasting for Picking

```typescript
import { useThree } from "@react-three/fiber";

function handleClick(
  event: MouseEvent,
  camera: THREE.Camera,
  scene: THREE.Scene,
  meshToExpressId: Map<THREE.Mesh, number>,
  ifcApi: IfcAPI,
  modelID: number
) {
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2(
    (event.clientX / window.innerWidth) * 2 - 1,
    -(event.clientY / window.innerHeight) * 2 + 1
  );
  raycaster.setFromCamera(mouse, camera);

  const intersects = raycaster.intersectObjects(Array.from(meshToExpressId.keys()));

  if (intersects.length > 0) {
    const hitMesh = intersects[0].object as THREE.Mesh;
    const expressID = meshToExpressId.get(hitMesh);

    if (expressID !== undefined) {
      const element = ifcApi.GetLine(modelID, expressID, true);
      console.log("Picked element:", element);

      // Highlight the selected element
      (hitMesh.material as THREE.MeshPhongMaterial).emissive.setHex(0x444444);
    }
  }
}
```

---

## 15. Filtering by IFC Type

web-ifc exports constants for every IFC entity type. Use these to filter which elements to process.

```typescript
import {
  IFCWALL, IFCWALLSTANDARDCASE, IFCSLAB, IFCCOLUMN, IFCBEAM,
  IFCDOOR, IFCWINDOW, IFCROOF, IFCSTAIR, IFCRAILING, IFCFURNISHINGELEMENT,
  IFCSPACE, IFCSITE, IFCBUILDING, IFCBUILDINGSTOREY,
  IFCOPENINGELEMENT, IFCFLOWSEGMENT, IFCFLOWFITTING
} from "web-ifc";

// Load only specific types
const typesToLoad = [IFCWALL, IFCSLAB, IFCCOLUMN, IFCBEAM, IFCDOOR, IFCWINDOW];

ifcApi.StreamAllMeshesWithTypes(modelID, typesToLoad, (mesh: FlatMesh) => {
  // Process only walls, slabs, columns, beams, doors, and windows
});

// Get all line IDs of a specific type (includeInherited = true to get subtypes)
const wallIds = ifcApi.GetLineIDsWithType(modelID, IFCWALL, true);
// This also returns IFCWALLSTANDARDCASE when includeInherited is true

// Check if an expressID is an IfcElement
const isElement = ifcApi.IsIfcElement(someTypeCode);

// Convert between type codes and names
const typeName = ifcApi.GetNameFromTypeCode(IFCWALL); // "IFCWALL"
const typeCode = ifcApi.GetTypeCodeFromName("IFCWALL"); // 3512223829
```

---

## 16. Coordination Matrix and Transformations

IFC files can contain coordinate offsets (e.g., real-world survey coordinates). Use the coordination matrix to handle this.

```typescript
// Get the 4x4 coordination matrix as a flat array of 16 numbers
const coordMatrix = ifcApi.GetCoordinationMatrix(modelID);
// Returns Float64Array(16) in column-major order

// Apply to Three.js
const matrix = new THREE.Matrix4().fromArray(coordMatrix);

// Set a custom geometry transformation for all subsequent geometry calls
ifcApi.SetGeometryTransformation(modelID, [
  1, 0, 0, 0,
  0, 1, 0, 0,
  0, 0, 1, 0,
  0, 0, 0, 1  // identity matrix
]);

// Get the world transform for a specific placement
const worldMatrix = ifcApi.GetWorldTransformMatrix(modelID, placementExpressID);
```

### Using COORDINATE_TO_ORIGIN

The simplest way to handle large coordinates:
```typescript
const modelID = ifcApi.OpenModel(data, {
  COORDINATE_TO_ORIGIN: true, // Automatically shifts model so geometry centers at origin
});
```

---

## 17. GUID / ExpressID Mapping

Every IFC element has a GloballyUniqueId (GUID). You can map between GUIDs and Express IDs:

```typescript
// Build the GUID → ExpressID mapping (call once after opening model)
ifcApi.CreateIfcGuidToExpressIdMapping(modelID);

// Look up an expressID from a GUID
const expressID = ifcApi.GetExpressIdFromGuid(modelID, "2O2Fr$t4X7Zf8NOew3FNr2");

// Look up a GUID from an expressID
const guid = ifcApi.GetGuidFromExpressId(modelID, 1234);

// Generate a new globally unique ID for new entities
const newGuid = ifcApi.CreateIFCGloballyUniqueId(modelID);
```

---

## 18. Streaming API (Memory-Efficient Large Files)

For large IFC files (100MB+), avoid `LoadAllGeometry()` and use streaming instead:

### Stream All Meshes

```typescript
ifcApi.StreamAllMeshes(modelID, (mesh: FlatMesh, index: number, total: number) => {
  // `mesh` is only valid during this callback!
  // Copy any data you need before the callback returns.

  const progress = ((index + 1) / total * 100).toFixed(1);
  console.log(`Processing ${progress}%`);

  const geometries = mesh.geometries;
  for (let i = 0; i < geometries.size(); i++) {
    const pg = geometries.get(i);
    // Process geometry...
  }
});
```

### Stream Specific Elements

```typescript
const idsToLoad = [100, 200, 300, 400]; // Express IDs
ifcApi.StreamMeshes(modelID, idsToLoad, (mesh: FlatMesh) => {
  // Process each mesh
});
```

### Stream by Type

```typescript
ifcApi.StreamAllMeshesWithTypes(
  modelID,
  [IFCWALL, IFCSLAB],
  (mesh: FlatMesh) => {
    // Only walls and slabs
  }
);
```

### Opening Files with Streaming Input

```typescript
// For very large files that don't fit in memory at once
const modelID = ifcApi.OpenModelFromCallback((offset, size) => {
  // Read `size` bytes starting at `offset` from your data source
  // Return a Uint8Array
  return myLargeFileReader.read(offset, size);
});
```

---

## 19. Multi-Threading

web-ifc ships with a multi-threaded WASM build (`web-ifc-mt.wasm`) that uses Web Workers and SharedArrayBuffer.

### Requirements

1. Your server must send these headers for multi-threading to work:
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

2. In Vite:
```typescript
// vite.config.ts
export default defineConfig({
  server: {
    headers: {
      "Cross-Origin-Opener-Policy": "same-origin",
      "Cross-Origin-Embedder-Policy": "require-corp",
    },
  },
});
```

### Usage

Multi-threading is automatic when:
- The browser supports `SharedArrayBuffer`
- The required HTTP headers are present
- The `web-ifc-mt.wasm` and `web-ifc-mt.worker.js` files are available

```typescript
// Multi-threading is used automatically, or you can force single-thread:
await ifcApi.Init(undefined, true); // forceSingleThread = true
```

---

## 20. Firebase Integration Patterns

### Store IFC Files in Firebase Storage

```typescript
import { getStorage, ref, uploadBytes, getDownloadURL, getBytes } from "firebase/storage";

// Upload
async function uploadIFC(modelID: number, projectId: string) {
  const data = ifcApi.SaveModel(modelID);
  const storage = getStorage();
  const fileRef = ref(storage, `projects/${projectId}/model.ifc`);
  await uploadBytes(fileRef, data);
  return await getDownloadURL(fileRef);
}

// Download and load
async function downloadAndLoadIFC(projectId: string): Promise<number> {
  const storage = getStorage();
  const fileRef = ref(storage, `projects/${projectId}/model.ifc`);
  const buffer = await getBytes(fileRef);
  return ifcApi.OpenModel(new Uint8Array(buffer), {
    COORDINATE_TO_ORIGIN: true,
  });
}
```

### Cache Extracted Properties in Firestore

IFC property extraction can be slow for large models. Cache results in Firestore:

```typescript
import { doc, setDoc, getDoc } from "firebase/firestore";

async function getOrCacheSpatialStructure(
  db: Firestore,
  projectId: string,
  ifcApi: IfcAPI,
  modelID: number
) {
  const cacheRef = doc(db, "projects", projectId, "cache", "spatialStructure");
  const cached = await getDoc(cacheRef);

  if (cached.exists()) {
    return cached.data();
  }

  // Extract and cache
  const structure = await ifcApi.properties.getSpatialStructure(modelID, true);
  await setDoc(cacheRef, structure);
  return structure;
}
```

### Store Element Metadata for Queries

```typescript
import { collection, writeBatch, doc } from "firebase/firestore";

async function indexElementsInFirestore(
  db: Firestore,
  projectId: string,
  ifcApi: IfcAPI,
  modelID: number
) {
  const batch = writeBatch(db);
  const elementsCol = collection(db, "projects", projectId, "elements");

  const allLines = ifcApi.GetAllLines(modelID);
  for (let i = 0; i < allLines.size(); i++) {
    const expressID = allLines.get(i);

    // Only index actual building elements
    const typeCode = ifcApi.GetLineType(modelID, expressID);
    if (!ifcApi.IsIfcElement(typeCode)) continue;

    const element = ifcApi.GetLine(modelID, expressID, false);
    const typeName = ifcApi.GetNameFromTypeCode(typeCode);

    batch.set(doc(elementsCol, String(expressID)), {
      expressID,
      type: typeName,
      name: element.Name?.value || null,
      globalId: element.GlobalId?.value || null,
    });

    // Firestore batches max out at 500 operations
    if ((i + 1) % 400 === 0) {
      await batch.commit();
    }
  }
  await batch.commit();
}
```

---

## 21. LoaderSettings Reference

Pass these as the second argument to `OpenModel()` or `CreateModel()`:

```typescript
interface LoaderSettings {
  /** Move model geometry to center on origin. Default: false */
  COORDINATE_TO_ORIGIN?: boolean;

  /** Number of segments for circle tessellation. Default: 12 */
  CIRCLE_SEGMENTS?: number;

  /** Maximum WASM memory in bytes. Default: 2147483648 (2GB) */
  MEMORY_LIMIT?: number;

  /** Internal tape buffer size. Default: 67108864 (64MB) */
  TAPE_SIZE?: number;

  /** Number of lines per write batch. Default: 10000 */
  LINEWRITER_BUFFER?: number;

  /** Tolerance for distance-based comparisons. Default: 1e-4 */
  TOLERANCE_DISTANCE?: number;

  /** Tolerance for line segments. Default: 1e-4 */
  TOLERANCE_LINE?: number;

  /** Tolerance for boolean operations. Default: 1e-4 */
  TOLERANCE_BOOLEAN?: number;

  /** Tolerance for scaling. Default: 1e-4 */
  TOLERANCE_SCALING?: number;

  /** Number of iterations for plane refitting. Default: 0 */
  PLANE_REFIT_ITERATIONS?: number;

  /** Number of solids before applying union. Default: 150 */
  BOOLEAN_UNION_THRESHOLD?: number;
}
```

---

## 22. Complete Type Reference

### Core Types

```typescript
/** Wrapper around WASM vectors. Use .size() and .get(i) to iterate. */
interface Vector<T> {
  size(): number;
  get(index: number): T;
}

/** A mesh for a single IFC element, may contain multiple geometries */
interface FlatMesh {
  expressID: number;
  geometries: Vector<PlacedGeometry>;
}

/** A positioned geometry instance within a FlatMesh */
interface PlacedGeometry {
  color: Color;
  geometryExpressID: number;
  flatTransformation: number[];  // 16 numbers, column-major 4x4 matrix
}

/** RGBA color, all channels 0-1 */
interface Color {
  x: number;  // Red
  y: number;  // Green
  z: number;  // Blue
  w: number;  // Alpha (1 = opaque)
}

/** Geometry data retrieved from WASM */
interface IfcGeometry {
  GetVertexData(): number;      // Pointer to vertex data
  GetVertexDataSize(): number;  // Size of vertex data in floats
  GetIndexData(): number;       // Pointer to index data
  GetIndexDataSize(): number;   // Size of index data in uint32s
  delete(): void;               // Free WASM memory (MUST call this!)
}

/** Raw line data from the IFC file */
interface RawLineData {
  ID: number;
  type: number;
  arguments: any[];
}

/** Configuration for creating a new model */
interface NewIfcModel {
  schema: string;           // "IFC2X3" | "IFC4" | "IFC4X3"
  name?: string;
  description?: string[];
  authors?: string[];
  organizations?: string[];
  authorization?: string;
}

/** IFC type information */
interface IfcType {
  typeID: number;
  typeName: string;
}

/** Reference to another IFC entity */
type Handle<T> = { value: number; type: 5 };
```

### Token Types

```typescript
const UNKNOWN = 0;
const STRING = 1;
const LABEL = 2;
const ENUM = 3;
const REAL = 4;
const REF = 5;
const EMPTY = 6;
const SET_BEGIN = 7;
const SET_END = 8;
const LINE_END = 9;
const INTEGER = 10;
```

### Header Types

```typescript
const FILE_DESCRIPTION = 0;
const FILE_NAME = 1;
const FILE_SCHEMA = 2;

// Usage:
const desc = ifcApi.GetHeaderLine(modelID, FILE_DESCRIPTION);
const name = ifcApi.GetHeaderLine(modelID, FILE_NAME);
const schema = ifcApi.GetHeaderLine(modelID, FILE_SCHEMA);
```

---

## 23. Complete API Method Reference

### IfcAPI — Initialization & Lifecycle

| Method | Signature | Description |
|--------|-----------|-------------|
| `Init` | `(customLocateFileHandler?, forceSingleThread?: boolean) => Promise<void>` | Initialize the WASM module. Must be called first. |
| `SetWasmPath` | `(path: string, absolute?: boolean) => void` | Set directory containing WASM files. Call before `Init()`. |
| `Dispose` | `() => void` | Close all models and shut down the WASM module. |
| `GetVersion` | `() => string` | Returns the web-ifc version string. |

### IfcAPI — Model Management

| Method | Signature | Description |
|--------|-----------|-------------|
| `OpenModel` | `(data: Uint8Array, settings?: LoaderSettings) => number` | Open an IFC file from buffer. Returns `modelID`. |
| `OpenModelFromCallback` | `(callback: (offset, size) => Uint8Array, settings?) => number` | Open IFC from streaming callback. |
| `OpenModels` | `(dataSets: Uint8Array[], settings?) => number` | Open multiple IFC files as one model. |
| `CreateModel` | `(model: NewIfcModel, settings?) => number` | Create a new empty IFC model. |
| `SaveModel` | `(modelID: number) => Uint8Array` | Export model as IFC STEP file bytes. |
| `SaveModelToCallback` | `(modelID: number, callback: (chunk) => void) => void` | Export via streaming callback. |
| `CloseModel` | `(modelID: number) => void` | Close model and free its memory. |
| `CloseAllModels` | `() => void` | Close all open models. |
| `IsModelOpen` | `(modelID: number) => boolean` | Check if a model is currently open. |
| `GetModelSchema` | `(modelID: number) => string` | Get IFC schema version (e.g., "IFC4"). |

### IfcAPI — Line Data (Read/Write/Delete)

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetLine` | `(modelID, expressID, flatten?, inverse?, inversePropKey?) => any` | Get a parsed IFC entity. `flatten=true` resolves nested refs. |
| `GetLines` | `(modelID, expressIDs[], flatten?, inverse?, inversePropKey?) => any[]` | Get multiple entities at once. |
| `GetAllLines` | `(modelID) => Vector<number>` | Get all express IDs in the model. |
| `GetLineIDsWithType` | `(modelID, type, includeInherited?) => Vector<number>` | Get IDs of all entities of a given type. |
| `GetRawLineData` | `(modelID, expressID) => RawLineData` | Get raw parsed data for one line. |
| `GetRawLinesData` | `(modelID, expressIDs[]) => RawLineData[]` | Get raw data for multiple lines. |
| `GetLineType` | `(modelID, expressID) => number` | Get the type code for a line. |
| `GetMaxExpressID` | `(modelID) => number` | Get the highest express ID. |
| `GetNextExpressID` | `(modelID, startID) => number` | Get next available ID after `startID`. |
| `WriteLine` | `(modelID, lineObject) => void` | Write/update a single entity. |
| `WriteLines` | `(modelID, lineObjects[]) => void` | Write/update multiple entities. |
| `DeleteLine` | `(modelID, expressID) => void` | Delete an entity from the model. |
| `FlattenLine` | `(modelID, line) => void` | Recursively resolve refs in a line object in-place. |

### IfcAPI — Entity Creation

| Method | Signature | Description |
|--------|-----------|-------------|
| `CreateIfcEntity` | `(modelID, type, ...args) => any` | Create a new IFC entity with given arguments. |
| `CreateIfcType` | `(modelID, type, value) => any` | Create a typed IFC value (e.g., IfcText, IfcReal). |
| `CreateIFCGloballyUniqueId` | `(modelID) => string` | Generate a new IFC GUID. |
| `CreateIfcGuidToExpressIdMapping` | `(modelID) => void` | Build internal GUID↔ID lookup table. |

### IfcAPI — Geometry

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetGeometry` | `(modelID, geometryExpressID) => IfcGeometry` | Get vertex/index data. **Must call `.delete()` after!** |
| `GetFlatMesh` | `(modelID, expressID) => FlatMesh` | Get mesh for a single element. |
| `LoadAllGeometry` | `(modelID) => Vector<FlatMesh>` | Load all meshes at once (high memory). |
| `StreamMeshes` | `(modelID, expressIDs[], callback) => void` | Stream meshes for specific IDs. |
| `StreamAllMeshes` | `(modelID, callback) => void` | Stream all element meshes. |
| `StreamAllMeshesWithTypes` | `(modelID, types[], callback) => void` | Stream meshes filtered by type. |
| `GetVertexArray` | `(ptr, size) => Float32Array` | Extract vertex data from WASM heap. |
| `GetIndexArray` | `(ptr, size) => Uint32Array` | Extract index data from WASM heap. |

### IfcAPI — Transforms & Coordinates

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetCoordinationMatrix` | `(modelID) => number[]` | Get 4x4 coordination matrix (16 floats). |
| `GetWorldTransformMatrix` | `(modelID, placementExpressID) => number[]` | Get world matrix for a placement. |
| `SetGeometryTransformation` | `(modelID, matrix: number[16]) => void` | Apply global transform to all geometry output. |

### IfcAPI — GUID Mapping

| Method | Signature | Description |
|--------|-----------|-------------|
| `CreateIfcGuidToExpressIdMapping` | `(modelID) => void` | Build GUID↔ID mapping. Must call before lookups. |
| `GetExpressIdFromGuid` | `(modelID, guid) => number` | Look up express ID from GUID string. |
| `GetGuidFromExpressId` | `(modelID, expressID) => string` | Look up GUID from express ID. |

### IfcAPI — Type Utilities

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetAllTypesOfModel` | `(modelID) => IfcType[]` | Get all IFC types present in the model. |
| `GetIfcEntityList` | `(modelID) => number[]` | Get all type codes in the model. |
| `GetNameFromTypeCode` | `(type) => string` | Convert type code to name (e.g., 3512223829 → "IFCWALL"). |
| `GetTypeCodeFromName` | `(name) => number` | Convert name to type code. |
| `IsIfcElement` | `(type) => boolean` | Check if type is a subtype of IfcElement. |

### IfcAPI — Header

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetHeaderLine` | `(modelID, headerType) => any` | Get FILE_DESCRIPTION (0), FILE_NAME (1), or FILE_SCHEMA (2). |

### IfcAPI — Misc

| Method | Signature | Description |
|--------|-----------|-------------|
| `SetLogLevel` | `(level: LogLevel) => void` | Set logging verbosity. |
| `EncodeText` | `(text: string) => string` | Encode text using IFC encoding rules. |
| `DecodeText` | `(text: string) => string` | Decode IFC-encoded text. |
| `ResetCache` | `(modelID) => void` | Clear cached geometry for the model. |

### Properties Helper (`ifcApi.properties`)

| Method | Signature | Description |
|--------|-----------|-------------|
| `getItemProperties` | `(modelID, id, recursive?, inverse?) => Promise<any>` | Get all properties of an element. |
| `getPropertySets` | `(modelID, elementID?, recursive?, includeType?) => Promise<any[]>` | Get property sets. |
| `setPropertySets` | `(modelID, elementID, psetID) => Promise<void>` | Associate a property set with an element. |
| `getTypeProperties` | `(modelID, elementID?, recursive?) => Promise<any[]>` | Get type object properties. |
| `getMaterialsProperties` | `(modelID, elementID?, recursive?, includeType?) => Promise<any[]>` | Get material info. |
| `setMaterialsProperties` | `(modelID, elementID, materialID) => Promise<void>` | Associate material with element. |
| `getSpatialStructure` | `(modelID, includeProperties?) => Promise<any>` | Get full spatial hierarchy tree. |

---

## 24. IFC Type Constants

web-ifc exports numeric constants for every IFC entity type. Import them directly:

```typescript
import {
  // Structural elements
  IFCWALL, IFCWALLSTANDARDCASE, IFCSLAB, IFCCOLUMN, IFCBEAM,
  IFCFOOTING, IFCPILE, IFCPLATE, IFCROOF, IFCSTAIR, IFCSTAIRFLIGHT,
  IFCRAMP, IFCRAMPFLIGHT, IFCRAILING, IFCMEMBER,

  // Openings
  IFCOPENINGELEMENT, IFCDOOR, IFCWINDOW,

  // Furnishing
  IFCFURNISHINGELEMENT, IFCFURNITURE,

  // MEP / Services
  IFCFLOWSEGMENT, IFCFLOWFITTING, IFCFLOWTERMINAL,
  IFCFLOWCONTROLLER, IFCFLOWSTORAGEDEVICE, IFCFLOWMOVINGDEVICE,
  IFCDISTRIBUTIONELEMENT,

  // Spatial
  IFCPROJECT, IFCSITE, IFCBUILDING, IFCBUILDINGSTOREY, IFCSPACE,

  // Geometry representations
  IFCEXTRUDEDAREASOLID, IFCBOOLEANCLIPPINGRESULT, IFCMAPPEDITEM,

  // Properties
  IFCPROPERTYSET, IFCPROPERTYSINGLEVALUE, IFCRELDEFINESBYPROPERTIES,
  IFCRELASSOCIATESMATERIAL, IFCRELDEFINESBYTYPE,

  // Relationships
  IFCRELAGGREGATES, IFCRELCONTAINEDINSPATIALSTRUCTURE,

  // Types for value creation
  IFCTEXT, IFCREAL, IFCINTEGER, IFCBOOLEAN, IFCLABEL,
  IFCIDENTIFIER, IFCGLOBALLYUNIQUEID, IFCLENGTHMEASURE,

  // ... hundreds more available
} from "web-ifc";
```

The full list is in `ifc-schema.ts` / `ifc-schema.d.ts`. Every IFC entity from IFC2X3 through IFC4x3 is available as a numeric constant.

---

## 25. Logging and Debugging

```typescript
import { LogLevel } from "web-ifc";

// Set log level before Init or anytime
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_DEBUG);   // Verbose, with stack traces
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_WARN);    // Warnings and errors
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_ERROR);   // Errors only (default)
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_OFF);     // Silent

// Check the library version
console.log("web-ifc version:", ifcApi.GetVersion());

// Inspect what IFC schema a model uses
console.log("Schema:", ifcApi.GetModelSchema(modelID)); // "IFC2X3", "IFC4", etc.

// List all entity types present in the model
const types = ifcApi.GetAllTypesOfModel(modelID);
types.forEach(t => console.log(`${t.typeName} (${t.typeID})`));
```

---

## 26. Performance Best Practices

1. **Use `StreamAllMeshes` instead of `LoadAllGeometry`** — streaming processes one mesh at a time and frees memory after each callback, while `LoadAllGeometry` loads everything into WASM memory at once.

2. **Always call `geometry.delete()`** — After extracting vertex/index data from `GetGeometry()`, call `.delete()` on the IfcGeometry to free WASM memory.

3. **Merge geometries for rendering** — Use Three.js `mergeGeometries()` to combine many small meshes into fewer draw calls. Separate opaque and transparent geometry.

4. **Use `COORDINATE_TO_ORIGIN: true`** — Prevents floating-point precision issues from large real-world coordinates.

5. **Increase `CIRCLE_SEGMENTS` only when needed** — Higher values (16-24) produce smoother curves but more geometry. Default 12 is usually fine.

6. **Close models when done** — `ifcApi.CloseModel(modelID)` frees all WASM heap memory for that model.

7. **Use `GetLineIDsWithType` with `includeInherited: true`** — This ensures you get subtypes (e.g., `IFCWALLSTANDARDCASE` when querying `IFCWALL`).

8. **Cache property data** — Reading properties via `GetLine(... , true)` is expensive for deeply nested data. Cache results rather than re-reading.

9. **Use `StreamAllMeshesWithTypes`** — If you only need walls and slabs, don't process all elements.

10. **Set `MEMORY_LIMIT` appropriately** — For very large files, increase from the 2GB default if your environment supports it.

---

## 27. Common Pitfalls

### WASM files not found
**Symptom:** `Init()` fails or hangs.
**Fix:** Ensure `.wasm` files are in your `public/` folder and `SetWasmPath()` points to the correct URL path. Check the browser Network tab.

### Geometry appears at wrong location
**Symptom:** Model is far from origin or invisible.
**Fix:** Use `COORDINATE_TO_ORIGIN: true` in `OpenModel` settings.

### Memory leak from geometry
**Symptom:** WASM memory grows continuously.
**Fix:** Always call `.delete()` on IfcGeometry objects returned by `GetGeometry()`.

### SharedArrayBuffer not available
**Symptom:** Multi-threading fails, error about SharedArrayBuffer.
**Fix:** Add COOP/COEP headers or use `forceSingleThread: true`.

### Empty geometry for some elements
**Symptom:** `GetFlatMesh` returns no geometries for certain IDs.
**Cause:** Not all IFC entities have geometric representations (e.g., IfcPropertySet, IfcRelAggregates). Only IfcElement subtypes have geometry.

### GetLine returns references instead of data
**Symptom:** Properties show `{value: 123, type: 5}` instead of actual data.
**Fix:** Use `flatten: true` parameter: `ifcApi.GetLine(modelID, id, true)`.

### Vector iteration
**Symptom:** Trying to use `for...of` or `.length` on Vector results.
**Fix:** web-ifc Vectors use `.size()` and `.get(i)`:
```typescript
const ids = ifcApi.GetAllLines(modelID);
for (let i = 0; i < ids.size(); i++) {
  const id = ids.get(i);
}
```

### React StrictMode double-initialization
**Symptom:** `Init()` called twice, errors or warnings.
**Fix:** Use a ref to track initialization state:
```typescript
const initialized = useRef(false);
useEffect(() => {
  if (initialized.current) return;
  initialized.current = true;
  ifcApi.Init().then(() => setReady(true));
}, []);
```

---

## 28. Updating to Future Versions

### Step-by-step Update Process

```bash
# 1. Check current version
npm list web-ifc

# 2. Check available versions
npm view web-ifc versions

# 3. Update to latest
npm install web-ifc@latest

# 4. CRITICAL: Re-copy WASM files to public/
cp node_modules/web-ifc/*.wasm public/
cp node_modules/web-ifc/*.worker.js public/

# 5. Verify the update
node -e "const w = require('web-ifc'); console.log('Installed:', Object.keys(w).length, 'exports')"
```

### What to Watch for in Updates

1. **WASM files must always be re-copied** after updating. The JS and WASM must be from the same version. Mismatched versions will cause cryptic runtime errors.

2. **Check the changelog** at [github.com/ThatOpen/engine_web-ifc/releases](https://github.com/ThatOpen/engine_web-ifc/releases) for breaking changes.

3. **API stability:** The core API (`OpenModel`, `GetLine`, `StreamAllMeshes`, etc.) has been stable across versions. Breaking changes are rare but possible in minor versions since the library is pre-1.0.

4. **Type definition changes:** After updating, run `tsc --noEmit` to check for type errors. The IFC schema constants may change between versions.

5. **Performance:** New versions often include geometry processing improvements. Re-benchmark if performance matters.

### Pinning a Version (recommended for production)

```json
{
  "dependencies": {
    "web-ifc": "0.0.76"
  }
}
```

Use an exact version (no `^` or `~`) to prevent unexpected updates. Update deliberately and test thoroughly.

### Automated WASM Copy Script

Add this to your `package.json` to ensure WASM files stay in sync:

```json
{
  "scripts": {
    "postinstall": "node -e \"const fs=require('fs'),p=require('path');const src='node_modules/web-ifc',dst='public';['web-ifc.wasm','web-ifc-mt.wasm','web-ifc-mt.worker.js'].forEach(f=>{const s=p.join(src,f);if(fs.existsSync(s))fs.copyFileSync(s,p.join(dst,f))});\""
  }
}
```

---

## Quick Reference: Minimal Working Example

```typescript
import { IfcAPI, IFCWALL } from "web-ifc";
import * as THREE from "three";

// 1. Init
const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/");
await ifcApi.Init();

// 2. Load
const response = await fetch("/model.ifc");
const data = new Uint8Array(await response.arrayBuffer());
const modelID = ifcApi.OpenModel(data, { COORDINATE_TO_ORIGIN: true });

// 3. Get geometry
const scene = new THREE.Scene();
ifcApi.StreamAllMeshes(modelID, (mesh) => {
  for (let i = 0; i < mesh.geometries.size(); i++) {
    const pg = mesh.geometries.get(i);
    const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);
    const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
    const indices = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

    // Build Three.js geometry from verts (6 floats per vertex: pos.xyz + norm.xyz)
    // ... (see Section 6 for full conversion code)

    geom.delete(); // Free WASM memory!
  }
});

// 4. Read properties
const walls = ifcApi.GetLineIDsWithType(modelID, IFCWALL, true);
for (let i = 0; i < walls.size(); i++) {
  const wall = ifcApi.GetLine(modelID, walls.get(i), true);
  console.log(wall.Name?.value);
}

// 5. Get spatial structure
const tree = await ifcApi.properties.getSpatialStructure(modelID);

// 6. Cleanup
ifcApi.CloseModel(modelID);
```

---

*This document was generated from web-ifc v0.0.76 source code at [ThatOpen/engine_web-ifc](https://github.com/ThatOpen/engine_web-ifc).*
