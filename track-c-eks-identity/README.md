# Pista C · IRSA contra EKS Pod Identity (opcional)

**Costo:** **sí cobra.** Un clúster de EKS se factura por hora, más el nodo y lo que consuma. Revisa la página de precios de Amazon EKS antes de empezar y **destruye todo al terminar** (el laboratorio completo debería tomar poco más de una hora). **Tiempo:** 60 a 75 minutos, de los cuales la creación del clúster es la mayor parte.

## Qué vas a comparar

Dos formas de darle credenciales de AWS a un pod, sin llaves guardadas:

| | IRSA | EKS Pod Identity |
|---|---|---|
| Confianza | El proveedor OIDC del clúster | El principal de servicio `pods.eks.amazonaws.com` |
| Variables que inyecta | `AWS_ROLE_ARN` y `AWS_WEB_IDENTITY_TOKEN_FILE` | `AWS_CONTAINER_CREDENTIALS_FULL_URI` y `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE` |
| Cómo se asocia | Anotación en la cuenta de servicio | Una *asociación* en la API de EKS |

AWS recomienda Pod Identity siempre que sea posible; IRSA sigue siendo la opción donde Pod Identity no está soportado.

## Requisitos

- Una cuenta de AWS **sandbox** con permisos para crear clústeres de EKS, roles de IAM y pilas de CloudFormation.
- `aws` CLI v2, [`eksctl`](https://eksctl.io/), `kubectl`.
- Los ejercicios de esta pista reutilizan los conceptos de la pista A (trust policies con `sub` y `aud`).

Para ver los comandos con tu ID de cuenta ya resuelto:

```bash
./track-c-eks-identity/scripts/render-cluster.sh
```

## C0 · Política de mínimo privilegio

Crea un parámetro de SSM y una política que solo permite leerlo (`ssm:GetParameter` sobre ese parámetro):

```bash
aws cloudformation deploy --stack-name trust-chain-lab-c \
  --template-file track-c-eks-identity/cfn/ssm-read-policy.yaml \
  --capabilities CAPABILITY_NAMED_IAM --region us-east-1
```

## C1 · Crea el clúster

```bash
eksctl create cluster -f track-c-eks-identity/cluster.yaml
```

Crea un clúster con Kubernetes 1.36, un nodo, el proveedor OIDC (`iam.withOIDC: true`) y el add-on `eks-pod-identity-agent`. Si en `us-east-1` aparece `UnsupportedAvailabilityZoneException`, repite el comando con `--zones` usando las zonas que sugiere el error.

## C2 · IRSA

```bash
kubectl create namespace lab
eksctl create iamserviceaccount --cluster trust-chain-lab --region us-east-1 \
  --namespace lab --name irsa-reader \
  --attach-policy-arn arn:aws:iam::<TU_CUENTA>:policy/trust-chain-lab-ssm-read --approve

kubectl apply -f track-c-eks-identity/manifests/pod-irsa.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/irsa-demo -n lab --timeout=120s
kubectl logs pod/irsa-demo -n lab
```

**Esperado:**
- Aparecen `AWS_ROLE_ARN` y `AWS_WEB_IDENTITY_TOKEN_FILE` (el webhook de EKS las inyecta por la anotación `eks.amazonaws.com/role-arn`).
- `sts get-caller-identity` muestra un ARN `assumed-role/...`.
- El parámetro de SSM se imprime.

Pregunta: ¿qué condiciones tiene la trust policy del rol que creó `eksctl`? Revísala en IAM (busca `:sub` y `:aud`).

## C3 · EKS Pod Identity

```bash
eksctl create podidentityassociation --cluster trust-chain-lab --region us-east-1 \
  --namespace lab --service-account-name pi-reader --create-service-account \
  --permission-policy-arns arn:aws:iam::<TU_CUENTA>:policy/trust-chain-lab-ssm-read

kubectl apply -f track-c-eks-identity/manifests/pod-podidentity.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/podidentity-demo -n lab --timeout=120s
kubectl logs pod/podidentity-demo -n lab
```

**Esperado:**
- Aparecen `AWS_CONTAINER_CREDENTIALS_FULL_URI` (hacia `169.254.170.23`) y `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE`; **no** aparecen las variables de IRSA.
- La misma salida de STS y de SSM que en C2.

Pregunta: la trust policy de este rol, ¿menciona el clúster? Compárala con la de C2.

```bash
eksctl get podidentityassociation --cluster trust-chain-lab --region us-east-1
```

## C4 · Una cuenta de servicio ajena

La trust policy de IRSA exige el `sub` de *una* cuenta de servicio. Prueba qué pasa con otra:

```bash
kubectl create serviceaccount intruder -n lab
ROLE_ARN=$(kubectl get sa irsa-reader -n lab -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')
kubectl annotate serviceaccount intruder -n lab eks.amazonaws.com/role-arn="$ROLE_ARN"

kubectl apply -f track-c-eks-identity/manifests/pod-irsa-intruder.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Failed pod/irsa-intruder -n lab --timeout=120s
kubectl logs pod/irsa-intruder -n lab
```

**Esperado:** el pod recibe las variables de IRSA, pero `get-caller-identity` falla porque el `sub` no coincide con el de `irsa-reader` (un error de autorización de `AssumeRoleWithWebIdentity`). Si **no** falla, revisa la trust policy: sin condición `sub`, cualquier cuenta de servicio del clúster podría asumir el rol.

## C5 · Migrar de IRSA a Pod Identity

Primero solo el **plan** (no cambia nada):

```bash
eksctl utils migrate-to-pod-identity --cluster trust-chain-lab --region us-east-1
```

Lee el plan: instala el agente si falta, agrega `pods.eks.amazonaws.com` a la trust policy de los roles de las cuentas de servicio y crea las asociaciones. Si quieres aplicarlo, agrega `--approve` y observa qué credenciales usa ahora el pod de IRSA (las que van antes en la cadena del SDK siguen ganando). Para quitar la confianza OIDC de esos roles existe `--remove-oidc-provider-trust-relationship`.

## C6 · Limpieza (obligatoria)

```bash
./track-c-eks-identity/scripts/down.sh
```

Borra el namespace, el clúster (con `--wait`) y la pila de CloudFormation. Luego revisa en la consola de CloudFormation que no queden pilas `eksctl-trust-chain-lab-*` y en IAM que no queden roles de esta pista.

## Para llevarte

- IRSA ata la confianza a **un clúster** y a una cuenta de servicio concreta; Pod Identity usa un principal de servicio único y las asociaciones viven en la API de EKS.
- Sin condición `sub`, la trust policy de IRSA abre el rol a cualquier cuenta de servicio del clúster.
- Ninguno de los dos cierra el acceso al rol del nodo por IMDS: eso se endurece aparte (IMDSv2 con límite de saltos 1).

## Puntos por validar

- Que `eksctl create iamserviceaccount` requiera o no `--approve` en tu versión de `eksctl`.
- El texto exacto del error de C4.
- El comportamiento de C5 con `--approve` cuando el rol confía en ambos mecanismos.
- Que `public.ecr.aws/aws-cli/aws-cli:latest` incluya `sh` (se asume por ser una imagen basada en Amazon Linux) y una versión de la CLI compatible con Pod Identity (se requiere 2.15.0 o posterior).
