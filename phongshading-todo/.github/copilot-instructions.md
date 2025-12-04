# Phong Shading & Texture Mapping - AI Development Guide

## Project Overview
A WebGL 2.0-based 3D graphics application implementing Phong illumination model with shadow mapping, cubemap skybox, and texture mapping. The application renders a cube and floor plane with dynamic lighting, interactive camera control, and real-time shadow computation.

## Architecture & Data Flow

### Core Components
- **`Phongshading.js`** - Main entry point; manages WebGL context, render loop, matrix transformations (Model/View/Projection), light properties, and camera control
- **`Models.js`** - Geometry generation (colorCube, plane); computes vertex positions, normals, and texture coordinates; stores in global arrays (points, normalsArray, texCoordsArray)
- **`configMaterialParameters.js`** - Phong material uniforms: ambient/diffuse/specular strength (all 0.5), shininess (100), light color (white)
- **`configTexture.js`** - Texture initialization: cubemap skybox (6 faces from `./skybox/`), 2D textures from PNG images (container.png, wood.png)

### Rendering Pipeline (Multi-Pass)
1. **Shadow Map Pass** - Renders depth from light's perspective to `framebuffer` using `depthProgram` (1024×1024 depth texture)
2. **Forward Rendering Pass** - Main scene rendering with `program`, inputs depth texture for shadow calculation, computes Phong illumination
3. **Skybox Pass** - Renders cubemap background using `skyboxProgram`
4. **Light Visualization** - Renders small cube marker at light source using `lampProgram` (optional debug aid)

### Key Global State Variables
- **Lighting**: `lightPosition` (vec4 with w={1:point, 0:directional}), `lightTheta`/`lightPhi`/`lightRadius` (spherical coords), `lightType`
- **Camera**: `eyePos`, `eyeRadius`/`eyeTheta`/`eyePhi` (spherical coords), `eyeFov` (55° default)
- **Matrices**: `ModelMatrix`, `ViewMatrix`, `ProjectionMatrix`, `lightSpaceMatrix`
- **Geometry counts**: `cubenumPoints`, `floornumPoints`, `skyboxnumPoints`, `lampnumPoints`

## Critical Patterns & Workflows

### Matrix Transformation Flow
```javascript
// Transform order: Projection × View × Model × vertex
gl_Position = u_ProjectionMatrix × u_ViewMatrix × u_ModelMatrix × vPosition;
// Normals require inverse-transpose for non-uniform scaling
Normal = transpose(inverse(mat3(u_ModelMatrix))) × vNormal.xyz;
```

### Geometry & Buffer Organization
- **Single VBO pattern**: All geometry (cube, floor, skybox) stored in one vertex array; indices track boundaries
- **Texture coordinates**: Cubemap faces use predefined `texCoord` array; 2D textures reuse same quad coordinates
- **Normal calculation**: Per-face normals computed as cross product in `quad()` function; duplicated per vertex

### Lighting Model (Phong in Fragment Shader)
- **Light direction detection**: `if(u_lightPosition.w == 1.0)` → point light (normalize direction vector), else directional light
- **Shadow map integration**: `shadowCalculation()` returns [0,1] opacity; TODO3 requires implementation
- **Texture sampling**: Diffuse color from 2D sampler; combines with Phong computation (ambient + diffuse + specular)

### Interactive Controls (Keyboard Event Handling)
- **C/V**: Toggle light type (point ↔ directional)
- **Y/U**: Increase/decrease `lightRadius`
- **W/S/A/D**: Rotate light around X/Y axes (modify `lightTheta`/`lightPhi`)
- **I/K/J/L/,/.**: Move camera (modify `eyePos`)
- **M/N**: Zoom projection (modify `eyeFov`)
- **Space**: Reset to `initParameters()` values

## Implementation Notes

### TODO Markers
- **TODO1** (`configTexture.js`): Complete 2D texture loading in `configureTexture()` function
- **TODO2** (likely in fragment shader): Integrate shadow calculation
- **TODO3** (`box.frag`): Implement `shadowCalculation()` - compare fragment depth to shadow map depth

### Framebuffer Setup for Shadow Mapping
- Depth texture (DEPTH_COMPONENT32F) replaces color attachment
- `gl.drawBuffers([gl.NONE])` and `gl.readBuffer([gl.NONE])` disable unused attachments
- Viewport changed to depth texture size (1024×1024) during shadow pass, restored afterward

### Canvas Viewport Configuration
```javascript
gl.viewport((canvas.width - canvas.height)/2, 0, canvas.height, canvas.height);
```
Creates square viewport centered horizontally; uses square canvas despite potentially wider window.

### Texture Unit Assignment
- **TEXTURE0**: Cubemap (skybox & main rendering)
- **TEXTURE1**: Diffuse map (cube & floor)
- **TEXTURE2**: Depth texture (shadow map)

## Common Development Tasks

### Adding a New Mesh
1. Create geometry function in `Models.js` (e.g., `sphere()`, `pyramid()`)
2. Track vertex count (update `numVertices`)
3. Push to global `points`, `colors`, `normalsArray`, `texCoordsArray`
4. In `Phongshading.js` window.onload, assign to variable (e.g., `spherenumPoints = sphere()`)
5. In render loop, call `gl.drawArrays(gl.TRIANGLES, startIndex, vertexCount)`

### Modifying Phong Parameters
- Material: Edit constants in `configurePhongModelMeterialParameters()`
- Light color: Modify `lightColor` variable (vec3 RGB)
- Shininess: Adjust `materialShininess` value

### Debugging Light & Shadows
- Print light position/matrix to console in render loop
- Visualize depth texture by binding to main rendering for inspection
- Enable `lampProgram` rendering to see light source location
- Check `u_LightSpaceMatrix` uniform transmission

## File Dependencies & Load Order
```
Phongshading.html loads:
  ├─ Common/webgl-utils.js (utility functions)
  ├─ Common/initShaders2.js (shader compilation)
  ├─ Common/MVnew.js (matrix/vector library)
  ├─ Models.js (geometry generation)
  ├─ configMaterialParameters.js (Phong uniforms)
  ├─ configTexture.js (texture initialization)
  └─ Phongshading.js (main application loop)
```
**Critical**: Load order must be maintained; `Phongshading.js` depends on all prior scripts.

## WebGL 2.0 Specifics
- GLSL ES 3.0 shaders required (`#version 300 es`)
- In/out variable syntax for vertex/fragment shaders
- `gl.drawArrays()` for non-indexed rendering (all meshes in single call with offset)
- Depth comparison uses `gl.LEQUAL` for standard depth testing
