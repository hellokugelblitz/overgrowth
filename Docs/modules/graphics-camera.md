# Graphics — Camera

Location:
- Source/Graphics/camera.h
- Source/Graphics/camera.cpp

Important responsibilities:
- view/projection matrices (GetPerspectiveMatrix, GetViewMatrix/GetProjMatrix)
- mouse ray and unproject utilities (GetMouseRay, UnProjectPixel, GetRayThroughPixel)
- frustum calculation and culling helpers (calcFrustumPlanes, checkSphereInFrustum, checkSpheresInFrustum, checkBoxInFrustum)
- interpolation state (SetInterpSteps, IncrementProgress, GetInterpWeight)

Common usage pattern:
1. Set camera state (pos, rotation, chase distance)
   camera.SetPos(...); camera.SetYRotation(...); camera.SetDistance(...);
2. On render frame call:
   camera.applyViewpoint(); // sets cameraViewMatrix, inverse, mouseray, frustum
3. To cull many spheres:
   - Prepare arrays float* x,y,z (see checkSpheresInFrustum signature)
   - Call with radius + cull_distance_squared
   - Read back uint32_t results (0 = not visible, 0xFFFFFFFF = visible)

Debugging:
- Print camera.GetPos(), camera.GetFacing() after applyViewpoint().
- Look at modelview/proj arrays (camera.GetViewMatrix(), GetProjMatrix()) before unProject calls.

Notes about the implementation:
- calcFrustumPlanes builds both scalar frustumPlanes[6][4] and SIMD copies (simdFrustumPlanes) used by checkSpheresInFrustum.
- Keep arrays 16-byte aligned if you add optimized SIMD culling code.
