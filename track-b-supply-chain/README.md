# Pista B · Cadena de suministro: firmar, verificar y bloquear

**Costo:** ninguno. Todo corre en GitHub Actions (repositorio público), GHCR y un clúster local con `kind`. **Tiempo:** 40 a 50 minutos.

## La idea

1. Un workflow construye una imagen y la **firma sin llaves** (cosign *keyless*): la identidad de la firma es el propio workflow, mediante un certificado de corta duración de Fulcio y una entrada en el registro de transparencia Rekor.
2. Un clúster local con **Kyverno** solo admite imágenes firmadas por *esa* identidad.

## Requisitos

- Tu **fork público** de este repositorio con Actions habilitado.
- Docker, [`kind`](https://kind.sigs.k8s.io/), `kubectl`, `helm` y [`cosign`](https://docs.sigstore.dev/) v3.
- Linux, macOS o WSL/Git Bash en Windows.

## B0 · Levanta el clúster

```bash
./track-b-supply-chain/scripts/up.sh
```

Crea un clúster `kind` con Kubernetes 1.36 (fijado a propósito: Kyverno 1.19 documenta pruebas con Kubernetes 1.33 a 1.36) e instala Kyverno 1.19.2.

## B1 · Construye y firma

1. En tu fork: *Actions → "B · Build, sign and push" → Run workflow* con `sign = true`.
2. En GitHub, abre el paquete que se creó (*Packages* en tu perfil, `.../demo`) y si quedó privado, cámbiale la visibilidad a **público** para que el clúster y Kyverno puedan leerlo sin credenciales.
3. Verifica la firma desde tu terminal. **Pon tus datos reales**, sin comodines:

```bash
cosign verify ghcr.io/TU_USUARIO/TU_FORK/demo:signed \
  --certificate-identity "https://github.com/TU_USUARIO/TU_FORK/.github/workflows/b-build-sign.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com"
```

Preguntas: ¿qué pasa si cambias el *issuer*? ¿Por qué verificar solo "que haya una firma" no basta?

## B2 · Instala la política y bloquea lo que no está firmado

```bash
./track-b-supply-chain/scripts/render-policy.sh TU_USUARIO TU_FORK
kubectl apply -f track-b-supply-chain/policies/verify-signed-images.yaml
```

Prueba con la imagen firmada:

```bash
kubectl run firmado --image=ghcr.io/TU_USUARIO_EN_MINUSCULAS/TU_FORK_EN_MINUSCULAS/demo:signed
```

**Esperado:** el pod se crea.

Ahora genera una imagen **sin firma**: ejecuta el workflow otra vez con `sign = false` y prueba:

```bash
kubectl run sinfirma --image=ghcr.io/TU_USUARIO_EN_MINUSCULAS/TU_FORK_EN_MINUSCULAS/demo:unsigned
```

**Esperado:** Kyverno rechaza el pod con el mensaje `Imagen sin firma válida de la identidad esperada`.

## B3 · La identidad importa

Edita `policies/verify-signed-images.yaml` y cambia `b-build-sign.yml` por otro nombre de workflow. Aplica de nuevo la política y vuelve a lanzar la imagen **firmada**.

**Esperado:** ahora también se rechaza, porque la firma es válida pero de *otra identidad*. Esa diferencia es la que se pierde cuando la verificación acepta cualquier identidad.

## B4 · Para pensar (sin comandos)

1. La política está en modo `Deny`. El ejemplo oficial usa `Audit`. ¿Qué cambia en la práctica?
2. Estás usando una **etiqueta** (`:signed`). ¿Qué riesgo tiene frente a un **digest**?
3. ¿Qué pasaría con `--certificate-identity-regexp '.*'`?
4. Esta firma vive en un registro de transparencia público: ¿qué datos tuyos quedan ahí?

## Limpieza

```bash
./track-b-supply-chain/scripts/down.sh
```

## Notas técnicas y puntos por validar

- Kyverno deprecó `ClusterPolicy` en la v1.19 y anunció su eliminación en la v1.20, por eso la pista usa `ImageValidatingPolicy`. Usa la v1.19.2, no la v1.20 candidata.
- Cosign 3.x usa el formato *bundle* por defecto y deprecó varios flags. Falta probar si Kyverno 1.19.2 verifica sin problemas firmas hechas con cosign 3.1.x.
- El registro Rekor público sigue en su versión 1 por defecto, según el blog de Sigstore.
- Esta pista no usa AWS. Para firmar imágenes dentro de AWS, mira la comparación entre Notation + AWS Signer y cosign en `docs/`.
