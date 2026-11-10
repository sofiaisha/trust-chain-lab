# Pista A · Identidad con IAM y OIDC (sin llaves estáticas)

**Costo:** ninguno (IAM, STS y SSM Parameter Store estándar no se cobran). **Tiempo:** 30 a 40 minutos.

## Requisitos

- Una cuenta de AWS **sandbox** y AWS CLI v2 configurada (esto es solo para desplegar el laboratorio desde tu terminal; el workflow no usa llaves).
- Tu propio **fork público** de este repositorio, con GitHub Actions habilitado. La trust policy se ata a *tu* repositorio.
- `curl` y `python3`.

## A0 · Averigua los IDs de tu repositorio

Desde 2026, los repositorios nuevos (o renombrados) emiten un claim `sub` **inmutable** que incluye los IDs del dueño y del repositorio:

```
repo:OWNER@OWNER-ID/REPO@REPO-ID:ref:refs/heads/BRANCH
```

```bash
./track-a-iam-oidc/scripts/get-repo-ids.sh TU_USUARIO tu-fork main
```

El script imprime el `sub` esperado y el comando de despliegue listo para copiar.

## A1 · Despliega el rol

Ejecuta el `aws cloudformation deploy` que imprimió el script. Crea el proveedor OIDC de GitHub (si ya existe en tu cuenta, agrega `CreateOidcProvider=false`), un parámetro de SSM y el rol `trust-chain-lab-gha`.

Lee `cfn/github-oidc-role.yaml` con calma y responde:

1. ¿Qué dos condiciones tiene la trust policy y por qué hacen falta las dos?
2. ¿Qué pasaría si el rol no tuviera la condición sobre `sub`? (Pista: IAM lo rechaza al crearlo.)
3. ¿Por qué `id-token: write` en el workflow no da permisos de escritura sobre nada?

## A2 · Haz que el workflow asuma el rol

1. Copia el `RoleArn` del *Output* del stack.
2. En tu fork: *Settings → Secrets and variables → Actions → Variables* y crea `AWS_ROLE_ARN`.
3. Ejecuta *Actions → "A · OIDC whoami" → Run workflow* en la rama `main`.

**Resultado esperado:** el paso "¿Quién soy?" muestra un ARN `assumed-role/trust-chain-lab-gha/trust-chain-lab` y el paso siguiente imprime `hola desde la cadena de confianza`. No creaste ni guardaste ninguna llave.

## A3 · Rompe y arregla

**Reto 1 · formato antiguo del `sub`.** Redespliega con el formato antiguo:

```bash
aws cloudformation deploy ... --parameter-overrides ... UseLegacySubFormat=true
```

Vuelve a ejecutar el workflow. **Esperado:** falla con un error del tipo "Not authorized to perform sts:AssumeRoleWithWebIdentity". Explica por qué (el repositorio es nuevo y GitHub ya emite el `sub` inmutable). Arregla con `UseLegacySubFormat=false`.

**Reto 2 · otra rama.** Cambia el parámetro `Branch` a `produccion` (sin crear esa rama) y ejecuta el workflow desde `main`. **Esperado:** falla. ¿Qué claim del token cambió respecto a lo que exige la trust policy?

## A4 · Valida trust policies con IAM Access Analyzer

En `challenges/` hay cuatro trust policies. Valida cada una (no crea nada):

```bash
aws accessanalyzer validate-policy \
  --policy-type RESOURCE_POLICY \
  --validate-policy-resource-type AWS::IAM::AssumeRolePolicyDocument \
  --policy-document file://track-a-iam-oidc/challenges/01-sin-sub.json
```

| Archivo | Qué tiene | Qué esperar (según la documentación) |
|---|---|---|
| `01-sin-sub.json` | Solo la condición `aud` | Un hallazgo de seguridad por falta de la condición del repositorio (`MISSING_GITHUB_REPO_CONDITION_KEY`) |
| `02-todo-el-dueno.json` | `sub` con comodín sobre todos los repos del dueño | Una advertencia sobre acotar a un repositorio y rama concretos |
| `03-repo-sin-rama.json` | `sub` del repo sin rama | Una advertencia que recomienda acotar la rama (`SPECIFIC_GITHUB_REPO_AND_BRANCH_RECOMMENDED`) |
| `04-correcta.json` | `sub` exacto con IDs y rama | Sin hallazgos de seguridad |

Anota el `issueCode` que realmente obtengas en cada caso: es parte del ejercicio y nos ayuda a corregir esta tabla.

## A5 · Encuentra el rastro en CloudTrail

Cada asunción deja un evento `AssumeRoleWithWebIdentity`. Búscalo en la región del workflow (`us-east-1`):

```bash
aws cloudtrail lookup-events --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --max-results 5 --query 'Events[].CloudTrailEvent' --output text
```

Busca en el JSON: `userIdentity.type`, el sujeto del token (`responseElements.subjectFromWebIdentityToken`), `responseElements.audience` y `additionalEventData.identityProviderConnectionVerificationMethod`. Los eventos pueden tardar unos minutos en aparecer.

## Limpieza

```bash
aws cloudformation delete-stack --stack-name trust-chain-lab-a --region us-east-1
```

Borra el rol, el parámetro y el proveedor OIDC (solo si lo creó este stack). Si dejaste un fork, ya no tiene acceso a nada en AWS.

## Para llevarte

- Sin condición de `sub` el rol es abierto para cualquier repositorio de GitHub.
- Los nombres de repositorio y usuario se pueden reciclar: por eso los IDs.
- Una trust policy válida puede aun así fallar en tiempo de ejecución si el formato del claim no coincide.
