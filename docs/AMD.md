# AMD base: plan de estabilización

Fecha: 2026-11-06. Aplica a `b3l0w-ort:amd-1.31`.

## 1. Estado actual

La base AMD compila con ROCm/MIGraphX hasta el provider, pero falla con:

```text
migraphx_execution_provider.cc:323: 'migraphx_shape_fp4x2_type' was not declared
migraphx_execution_provider.cc:1261: 'quantize_bf16' is not a member of 'migraphx'
```

No es un flag faltante. ORT 1.31 usa APIs de MIGraphX que **no existen en MIGraphX 2.12** (ROCm 6.4).
Los intentos anteriores ya resolvieron, en orden:

1. `migraphx-dev` faltante.
2. `miopen-hip-dev`, `rocblas-dev`, `hipblaslt-dev`, `half` faltantes.
3. `migraphx/version.h` en `/opt/rocm/lib/migraphx/include`, fuera del include path de ORT.

El bloqueo restante es de **versión de MIGraphX**, no de empaquetado.

## 2. Decisión

Mantener ORT 1.31 y subir el builder a **ROCm 7.2**, que trae MIGraphX 2.13/MIOpen 3.5. No parchear
ORT para 2.12: los símbolos son parte del provider y un `#ifdef` casero sería deuda frágil.

```text
rocm/dev-ubuntu-24.04:7.2   builder
rocm runtime 7.2            base publicada
```

Si 7.2 tampoco expone los símbolos, las alternativas están en §7.

## 3. Preflight obligatorio (2–5 min, sin compilar ORT)

Antes de lanzar un build de 40–60 min, validar **símbolos**, no solo `find_package`:

```sh
docker run --rm rocm/dev-ubuntu-24.04:7.2 bash -euxo pipefail -c '
  apt-get update
  apt-get install -y --no-install-recommends \
    cmake git migraphx-dev miopen-hip-dev rocblas-dev hipblaslt-dev half
  mkdir -p /opt/rocm/include/migraphx
  ln -sf /opt/rocm/lib/migraphx/include/migraphx/version.h \
        /opt/rocm/include/migraphx/version.h
  grep -R "migraphx_shape_fp4x2_type" /opt/rocm/include /opt/rocm/lib/migraphx/include
  grep -R "quantize_bf16" /opt/rocm/include /opt/rocm/lib/migraphx/include
'
```

Además:

- `migraphx-config.cmake` debe resolver `MIOpen`, `rocblas`, `hipblaslt`.
- `find_package(hip|migraphx)` con `-DCMAKE_PREFIX_PATH="/opt/rocm;/opt/rocm/hip"`.
- `build.sh --help | grep compile_no_warning_as_error`.

**Regla:** si un grep de símbolos falla, no se compila. Se cambia de imagen o se documenta el bloqueo.

## 4. Preflight de CMake

```cmake
cmake_minimum_required(VERSION 3.20)
project(preflight CXX)
find_package(Threads REQUIRED)
find_package(hip REQUIRED)
find_package(migraphx REQUIRED)
find_package(MIOpen REQUIRED)
find_package(rocblas REQUIRED)
find_package(hipblaslt REQUIRED)
message(STATUS "AMD preflight OK")
```

Se corre con:

```sh
cmake -S /tmp/preflight -B /tmp/preflight/build \
  -DCMAKE_PREFIX_PATH="/opt/rocm;/opt/rocm/hip"
```

## 5. Build

```sh
./build.sh --config Release --build_shared_lib --parallel --skip_tests \
  --skip_submodule_sync --use_migraphx --migraphx_home /opt/rocm \
  --compile_no_warning_as_error \
  --allow_running_as_root --build_dir /build
```

Validar antes de publicar:

```sh
ls -lh /build/Release/libonnxruntime.so*
ls -lh /build/Release/libonnxruntime_providers_migraphx.so
ldd /build/Release/libonnxruntime_providers_migraphx.so | grep -i "not found" && exit 1 || true
```

La base final **no** lleva toolchain: copia solo `libonnxruntime*.so` y el provider; instala
`migraphx`, `miopen-hip`, `half` y las libs runtime.

## 6. Presupuesto de runner

- Imagen `rocm/dev 7.2`: más grande que 6.4 (6.4 uncompressed = 3.98 GB). Medir antes de asumir.
- `free-disk-space` deja ~43 GB.
- `docker buildx build --push` evita guardar la imagen final en el runner.
- Un job exclusivo para AMD; no compartir con Intel.
- Timeout 360 min.
- Si el disco no entra: construir en local/server y pushear la base; no es un fracaso del código.

## 7. Si ROCm 7.2 no alcanza

Orden de fallback:

1. **ROCm 7.2 + MIGraphX 2.13/2.14** (primera opción).
2. **ROCM 7.x con MIGraphX develop**: solo si el paquete estable no expone los símbolos.
3. **Parche mínimo de ORT** con `#ifdef` por versión de MIGraphX: último recurso; hay que mantenerlo
   en cada actualización de ORT.
4. **Plugin EP `onnxruntime-ep-amdgpu`**: evita compilar ORT entero, pero el binding Go necesita
   registrar el plugin; no es el camino inicial.
5. **ORT 1.23.2 + ROCm 6.4** para AMD: incompatible con la base única de Argos/Sonus; solo si se acepta
   un runtime separado.

## 8. Validación

- `ldd` del provider sin `not found`.
- `argosd` con un modelo descargado y **CPU EP** primero: debe arrancar y responder `/readyz`.
- En hardware AMD: `doctor` debe reportar `MIGraphX` como provider real; una búsqueda debe ejecutar
  sin fallback silencioso.
- Sin GPU AMD no se puede declarar “acelerado”; sí “compila y carga”.

## 9. Criterios de parada

- Preflight de símbolos falla → cambiar imagen, no compilar.
- Compile falla en un archivo concreto → guardar log y símbolo; no reintentar la misma combinación.
- Link falla → revisar `migraphx::c`, `hip::host`, `MIOpen`, `rocblas`, `hipblaslt`.
- Runtime falla al cargar provider → revisar `LD_LIBRARY_PATH`, versión de MIGraphX en runtime y
  `migraphx-driver --version`.
