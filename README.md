# b3l0w-ort

Imágenes base de runtime + ONNX Runtime para las apps de b3l0w. **No contienen binarios de
Argos, Auris ni Vox** y no llevan modelos.

## Variantes

```text
ghcr.io/below-z3r0/b3l0w-ort:cpu-1.31
ghcr.io/below-z3r0/b3l0w-ort:cuda-1.31
ghcr.io/below-z3r0/b3l0w-ort:intel-1.31
ghcr.io/below-z3r0/b3l0w-ort:amd-1.31
```

| Variante | Base | ORT |
|---|---|---|
| cpu | Debian 13 slim | CPU |
| cuda | nvidia/cuda 12.8.1-cudnn-runtime | CUDA oficial |
| intel | OpenVINO runtime 2026.4.1 | OpenVINO, compilado |
| amd | ROCm 6.4 | MIGraphX, compilado |

## Build

```sh
docker build -f docker/base.Dockerfile --target base-cpu -t b3l0w-ort:cpu-1.31 .
docker build -f docker/base.Dockerfile --target base-cuda -t b3l0w-ort:cuda-1.31 .
docker build -f docker/base.Dockerfile --target base-intel -t b3l0w-ort:intel-1.31 .
docker build -f docker/base.Dockerfile --target base-amd -t b3l0w-ort:amd-1.31 .
```

El workflow `.github/workflows/ort.yml` hace las cuatro variantes en matrix. Cada variante corre en
su propia VM; no se borra nada a mano: el runner se destruye al terminar el job.

## Consumo desde una app

```dockerfile
ARG BASE_IMAGE=ghcr.io/below-z3r0/b3l0w-ort:cuda-1.31
FROM ${BASE_IMAGE}
COPY --from=build /out/ /usr/local/bin/
```

La app debe incluir su propio entrypoint; la base deja `tini`, usuario `argos`, `/home/argos` y
`ARGOS_ORT_LIB` como convención compartida.

## Cambios

- Subir `ORT_VERSION` o cambiar OpenVINO/ROCm obliga a reconstruir la base.
- Un cambio de app **no** toca este repo.
- Los tags de app deben declarar con qué base se construyeron (label o digest en el manifiesto).
