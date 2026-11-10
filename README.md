# trust-chain-lab

Laboratorio práctico de la charla **"De commit a pod: una cadena de confianza sin un solo secreto"** (ROXS.FEST 2026, 12 de noviembre de 2026).

> Estado: **borrador en construcción**. Los ejercicios se escribieron a partir de documentación oficial y todavía hay que validarlos en una cuenta sandbox de AWS y en un clúster local. No uses nada de aquí en producción sin revisarlo.

## Qué vas a practicar

| Pista | Qué haces | Costo | Requiere |
|---|---|---|---|
| **A. Identidad (IAM + OIDC)** | Federar GitHub Actions con AWS mediante OIDC y STS, sin llaves estáticas. Romper y arreglar trust policies. | Sin costo | Una cuenta de AWS sandbox y AWS CLI |
| **B. Cadena de suministro** | Firmar una imagen con cosign (keyless) en GitHub Actions y bloquear en el clúster lo que no esté firmado con Kyverno. | Gratis | Docker, `kind`, `kubectl`, `helm` y `cosign` |
| **C. Identidad en Kubernetes (opcional)** | Comparar IRSA y EKS Pod Identity en un clúster real de EKS. | **Cobra por hora**: destrúyelo al terminar | Cuenta sandbox de AWS, `eksctl` y `kubectl` |

## Reglas del lab

1. Usa **solo una cuenta sandbox**, nunca producción.
2. Cada pista incluye su limpieza: ejecútala al terminar.
3. Ningún ejercicio pide llaves de acceso de larga duración. Si ves una, es un error del lab: abre un issue.
4. Haz **fork público** de este repositorio: las identidades (el `sub` de GitHub y la identidad de la firma) se atan a *tu* repositorio.

## Estructura

```
track-a-iam-oidc/        # IAM, STS y federación OIDC (CloudFormation + workflow + retos)
track-b-supply-chain/    # Firma, verificación y política en un clúster local
track-c-eks-identity/    # IRSA contra EKS Pod Identity (opcional; cobra por hora)
docs/                    # Estándares y fuentes verificadas
.github/workflows/       # Workflows de las pistas A y B
```

## Estándares y fuentes

Ver [`docs/estandares-y-fuentes.md`](docs/estandares-y-fuentes.md): AWS Security Reference Architecture, guía de buenas prácticas de EKS, SLSA, Sigstore, Kyverno, NIST SSDF y la documentación de IAM/STS, con las fechas de los cambios recientes.

## Autoría y licencia

Creado por Sofy Romero. Licencia Apache-2.0 (ver el archivo [`LICENSE`](LICENSE)). Las opiniones son personales.
