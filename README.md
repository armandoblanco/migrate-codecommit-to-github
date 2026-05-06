# Migracion de AWS CodeCommit a GitHub

## Contexto

AWS CodeCommit esta en proceso de deprecacion (no acepta nuevos clientes desde julio 2024). Esta guia documenta el proceso para migrar repositorios de CodeCommit a GitHub y reconfigurar los pipelines de AWS CodePipeline para que sigan funcionando con GitHub como fuente, resolviendo las dudas de seguridad y aislamiento del AWS Connector for GitHub.

## Preguntas frecuentes y respuestas tecnicas

### 1. Como asegurar la conexion GitHub a AWS para que CodePipeline se active con cada commit

La integracion se hace a traves de **AWS CodeConnections** (antes llamado CodeStar Connections). Cuando configuras un pipeline con el provider `CodeStarSourceConnection`, AWS instala la GitHub App "AWS Connector for GitHub" en tu organizacion de GitHub. Esta app recibe webhooks automaticamente: cada push a la rama configurada dispara el pipeline sin necesidad de configurar webhooks manualmente.

La configuracion en el pipeline se ve asi:

```yaml
Configuration:
  ConnectionArn: "arn:aws:codeconnections:REGION:ACCOUNT_ID:connection/CONNECTION_ID"
  FullRepositoryId: "ORG/REPO"
  BranchName: "main"
  OutputArtifactFormat: "CODE_ZIP"
```

Tambien puedes configurar triggers mas granulares para que el pipeline se active solo con push a ramas especificas o con tags especificos, usando la seccion de Pipeline Triggers en la definicion del pipeline.

**Lo que necesitas del lado de IAM** es que el service role de CodePipeline tenga permiso `codeconnections:UseConnection` sobre el ARN de la conexion:

```json
{
  "Effect": "Allow",
  "Action": "codeconnections:UseConnection",
  "Resource": "arn:aws:codeconnections:REGION:ACCOUNT_ID:connection/CONNECTION_ID"
}
```

> **Nota sobre prefijos**: Si la conexion se creo antes de julio 2024, el ARN usa el prefijo `codestar-connections`. Las conexiones nuevas usan `codeconnections`. El permiso IAM debe coincidir con el prefijo del ARN del recurso. En caso de duda, incluye ambos prefijos en la politica.

### 2. El AWS Connector de GitHub: riesgo de seguridad como conector unico

**Si, es un problema real y documentado.** No te voy a endulzar esto.

El "AWS Connector for GitHub" es una GitHub App que se instala a nivel de organizacion de GitHub. Cuando la instalas, le otorgas acceso a repositorios especificos (o a todos). El problema fundamental es el siguiente:

Cada cuenta de AWS que crea una CodeConnection contra la misma organizacion de GitHub comparte la misma GitHub App installation. Esto significa que **la CodeConnection en cada cuenta de AWS hereda todos los permisos que la GitHub App tiene en la organizacion**, no solo los repos que esa cuenta necesita.

Puesto de otra forma: si la GitHub App tiene acceso a 100 repos y tu cuenta de AWS solo necesita 5, la CodeConnection de tu cuenta tecnicamente puede llegar a los 100 si alguien con privilegios suficientes en AWS crea un IAM role sin restricciones.

Hay investigacion reciente (Thomas Preece, diciembre 2025 / marzo 2026) que demuestra que desde un CodeBuild job es posible extraer el token raw de la GitHub App via una API no documentada, y ese token tiene permisos de lectura, escritura y **administracion** sobre todos los repos a los que la App tiene acceso. Esto incluye la capacidad de deshabilitar branch protections.

**Mitigaciones:**

La mitigacion del lado de AWS se hace con **condition keys** en IAM. La accion `codeconnections:UseConnection` soporta las siguientes condition keys:

| Condition Key | Descripcion |
|---|---|
| `codeconnections:FullRepositoryId` | Restringe a un repositorio especifico (formato `org/repo`) |
| `codeconnections:BranchName` | Restringe a una rama especifica |
| `codeconnections:RepositoryName` | Restringe por nombre de repositorio |
| `codeconnections:OwnerId` | Restringe por owner del repositorio |
| `codeconnections:ProviderAction` | Restringe que operaciones se pueden hacer (read, clone, push, etc.) |
| `codeconnections:ProviderPermissionsRequired` | Restringe a `read_only` o `read_write` |

Ejemplo de politica IAM que restringe el service role de un pipeline a un solo repositorio:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "codeconnections:UseConnection",
      "Resource": "arn:aws:codeconnections:REGION:ACCOUNT_ID:connection/CONNECTION_ID",
      "Condition": {
        "StringEquals": {
          "codeconnections:FullRepositoryId": "mi-org/repo-proyecto-x"
        }
      }
    }
  ]
}
```

**Pero** estas condition keys solo funcionan a nivel de `UseConnection`. La API `GetConnectionToken` (que es la que usa CodeBuild internamente para obtener el token Git) **no tiene condition keys**. Esto significa que la restriccion por repositorio aplica para CodePipeline, pero un actor con acceso a CodeBuild podria evadir estas restricciones.

### 3. Desde una cuenta de AWS se pueden ver repos de otros proyectos si el conector es compartido?

**Si.** Si multiples cuentas de AWS crean CodeConnections contra la misma organizacion de GitHub, y la GitHub App tiene acceso a repos de diferentes proyectos/equipos, entonces tecnicamente una cuenta de AWS puede acceder a repos que pertenecen a otro equipo.

Esto pasa porque la CodeConnection hereda los permisos completos de la GitHub App installation, y AWS no permite restringir la CodeConnection durante la creacion. La unica restriccion disponible es la que apliques despues via IAM condition keys, y como se explico arriba, esa restriccion tiene limites.

### 4. Como asegurar que la conexion de la cuenta X solo vea sus repositorios

Hay tres niveles de estrategia, del mas seguro al menos seguro:

**Opcion A: Una GitHub App installation separada por equipo/cuenta (recomendada para aislamiento fuerte)**

En lugar de instalar una sola GitHub App para toda la organizacion, puedes instalar la "AWS Connector for GitHub" multiples veces, cada una con acceso unicamente a los repos que necesita un equipo. Cada cuenta de AWS crea su CodeConnection apuntando a su propia installation.

La limitacion es que esto solo funciona si los repos estan bajo organizaciones de GitHub diferentes, o si usas la configuracion de "Repository Access" de la GitHub App para segmentar que repos puede ver cada installation. AWS documenta que para trabajar con multiples organizaciones en el mismo servidor GitHub, debes crear conexiones separadas por organizacion.

**Opcion B: Una sola GitHub App pero con IAM condition keys estrictos**

Si no puedes separar las installations (por ejemplo, todos los repos estan en una sola org de GitHub), aplica condition keys en cada IAM role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictToProjectXRepos",
      "Effect": "Allow",
      "Action": "codeconnections:UseConnection",
      "Resource": "arn:aws:codeconnections:REGION:ACCOUNT_ID:connection/CONNECTION_ID",
      "Condition": {
        "StringEquals": {
          "codeconnections:FullRepositoryId": [
            "mi-org/proyecto-x-api",
            "mi-org/proyecto-x-frontend",
            "mi-org/proyecto-x-infra"
          ]
        }
      }
    },
    {
      "Sid": "DenyCodeBuildWithConnection",
      "Effect": "Deny",
      "Action": "codeconnections:GetConnectionToken",
      "Resource": "*"
    }
  ]
}
```

El `Deny` en `GetConnectionToken` es una recomendacion para mitigar el vector de ataque via CodeBuild. Solo otorga ese permiso a roles de CodeBuild que realmente lo necesiten.

**Opcion C: Conexion compartida via AWS RAM (mas conveniente, menos segura)**

AWS Resource Access Manager permite compartir una CodeConnection entre cuentas. Es mas facil de administrar pero no cambia los permisos que cada cuenta obtiene sobre los repos de GitHub. Solo centraliza la gestion.

## Parte 1: Migracion de repositorios

```
Fase 1: Prerrequisitos        -->   Herramientas, accesos, GitHub App
Fase 2: Clonar de CodeCommit  -->   git clone --mirror desde AWS
Fase 3: Escaneo de secretos   -->   gitleaks + limpieza si es necesario
Fase 4: Push a GitHub         -->   Push con historial completo
Fase 5: Configurar pipeline   -->   CodePipeline apuntando a GitHub
```

### Fase 1: Prerrequisitos

**Herramientas necesarias:**

Se necesita tener instalado: Git 2.x+, AWS CLI v2, GitHub CLI (`gh`), y gitleaks.

**Accesos necesarios:**

Del lado de AWS se necesitan credenciales HTTPS Git para CodeCommit (se generan desde IAM > Users > Security Credentials > HTTPS Git credentials for CodeCommit), o bien un perfil de AWS CLI con permisos `codecommit:GitPull`. Del lado de GitHub se necesita un PAT con scope `repo` o acceso SSH configurado, y permisos de admin en la organizacion de GitHub destino.

**Crear la CodeConnection:**

```bash
# Crear la conexion (quedara en estado PENDING)
aws codeconnections create-connection \
  --provider-type GitHub \
  --connection-name "codecommit-migration" \
  --region REGION
```

Despues de crear la conexion, hay que completar el handshake desde la consola de AWS (Developer Tools > Settings > Connections). Esto abre un flujo OAuth donde el owner de la organizacion de GitHub autoriza la GitHub App y selecciona a que repos tiene acceso.

**Punto critico:** En este paso es donde decides el alcance de la GitHub App. Si quieres aislamiento por equipo, selecciona SOLO los repos que necesita esa cuenta de AWS. No selecciones "All repositories".

### Fase 2: Clonar desde CodeCommit

```bash
# Clonar con mirror (incluye todas las ramas, tags y refs)
git clone --mirror https://git-codecommit.REGION.amazonaws.com/v1/repos/MI-REPO
cd MI-REPO.git

# Verificar que el clone esta completo
git branch -a
git tag -l
```

Si los repos usan Git LFS:

```bash
# Antes del clone
git lfs install
git clone --mirror https://git-codecommit.REGION.amazonaws.com/v1/repos/MI-REPO
cd MI-REPO.git
git lfs fetch --all
```

### Fase 3: Escaneo de secretos

**Esto no es opcional.** CodeCommit probablemente no tenia GitHub Secret Protection habilitado. Antes de hacer push a GitHub hay que verificar que no estas migrando secretos expuestos en el historial.

```bash
# Instalar gitleaks si no lo tienes
# macOS: brew install gitleaks
# Linux: ver releases en github.com/gitleaks/gitleaks

# Escanear todo el historial
gitleaks detect --source . --verbose --report-format json --report-path gitleaks-report.json
```

Si gitleaks encuentra secretos:

```bash
# Opcion 1: Limpiar con BFG (si los secretos estan en commits viejos)
# Descargar BFG: https://rtyley.github.io/bfg-repo-cleaner/
java -jar bfg.jar --replace-text passwords.txt MI-REPO.git
cd MI-REPO.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Opcion 2: Si el secreto esta en el ultimo commit, simplemente rotalo
# y actualiza el archivo antes de hacer push
```

**Despues de limpiar, vuelve a escanear para confirmar:**

```bash
gitleaks detect --source . --verbose
```

### Fase 4: Push a GitHub

```bash
# Crear el repo en GitHub (privado por defecto)
gh repo create MI-ORG/MI-REPO --private --confirm

# Agregar remote y push
cd MI-REPO.git
git remote add github git@github.com:MI-ORG/MI-REPO.git

# Push todo: ramas + tags
git push github --mirror

# Si usas LFS
git lfs push github --all
```

**Verificacion post-push:**

```bash
# Verificar que las ramas llegaron
gh api repos/MI-ORG/MI-REPO/branches --jq '.[].name'

# Verificar tags
gh api repos/MI-ORG/MI-REPO/tags --jq '.[].name'

# Verificar conteo de commits en rama principal
gh api repos/MI-ORG/MI-REPO/commits?per_page=1 -i | grep -i "link:"
```

### Fase 5: Reconfigurar CodePipeline

Hay dos caminos: actualizar el pipeline existente o crear uno nuevo. Si el pipeline tiene multiples stages complejos, actualizar es mas seguro.

**Actualizar pipeline existente via CLI:**

```bash
# Exportar la definicion actual
aws codepipeline get-pipeline --name MI-PIPELINE > pipeline.json

# Editar pipeline.json:
# 1. Cambiar el source stage de CodeCommit a CodeStarSourceConnection
# 2. Reemplazar la configuracion del source

# El source stage debe quedar asi:
# {
#   "name": "Source",
#   "actions": [{
#     "name": "SourceAction",
#     "actionTypeId": {
#       "category": "Source",
#       "owner": "AWS",
#       "provider": "CodeStarSourceConnection",
#       "version": "1"
#     },
#     "configuration": {
#       "ConnectionArn": "arn:aws:codeconnections:REGION:ACCOUNT:connection/ID",
#       "FullRepositoryId": "MI-ORG/MI-REPO",
#       "BranchName": "main",
#       "OutputArtifactFormat": "CODE_ZIP"
#     },
#     "outputArtifacts": [{"name": "SourceArtifact"}]
#   }]
# }

# Quitar el campo "metadata" del JSON (codepipeline no lo acepta en update)
# Actualizar
aws codepipeline update-pipeline --cli-input-json file://pipeline.json
```

**Actualizar el service role de CodePipeline:**

```bash
# El service role necesita estos permisos adicionales:
# - codeconnections:UseConnection (sobre el ARN de la conexion)
# Si el role fue creado antes de diciembre 2019, agregar el permiso explicitamente

aws iam put-role-policy \
  --role-name CodePipelineServiceRole \
  --policy-name CodeConnectionAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "codeconnections:UseConnection",
      "Resource": "arn:aws:codeconnections:REGION:ACCOUNT_ID:connection/CONNECTION_ID",
      "Condition": {
        "StringEquals": {
          "codeconnections:FullRepositoryId": "MI-ORG/MI-REPO"
        }
      }
    }]
  }'
```

### Fase 6: Validacion

```bash
# Trigger manual para verificar que funciona
aws codepipeline start-pipeline-execution --name MI-PIPELINE

# Monitorear ejecucion
aws codepipeline get-pipeline-execution \
  --pipeline-name MI-PIPELINE \
  --pipeline-execution-id EXECUTION_ID

# Hacer un commit de prueba en GitHub y verificar que el pipeline arranca
echo "# test" >> test-trigger.md
git add test-trigger.md
git commit -m "test: validar trigger de CodePipeline"
git push origin main
```

## Parte 2: Estrategia de aislamiento multi-cuenta

### Escenario tipico

Multiples cuentas de AWS (Cuenta-A, Cuenta-B, Cuenta-C), cada una con sus propios pipelines, y una sola organizacion de GitHub con repositorios de todos los equipos.

### Estrategia recomendada

```
Organizacion GitHub (mi-org)
|
+-- GitHub App "AWS Connector" (installation)
|   +-- Repos permitidos: seleccion explicita por equipo
|
+-- Cuenta AWS A
|   +-- CodeConnection A (ConnectionArn A)
|   +-- IAM Role Pipeline A
|       +-- Condition: FullRepositoryId = ["mi-org/repo-a1", "mi-org/repo-a2"]
|
+-- Cuenta AWS B
|   +-- CodeConnection B (ConnectionArn B)
|   +-- IAM Role Pipeline B
|       +-- Condition: FullRepositoryId = ["mi-org/repo-b1", "mi-org/repo-b2"]
```

Para aislamiento fuerte, lo ideal es que cada cuenta de AWS tenga su propia CodeConnection completada por el GitHub org owner, y que la GitHub App se configure con "Only select repositories" apuntando exclusivamente a los repos de ese equipo. Si necesitas que la GitHub App tenga acceso a repos de multiples equipos (porque es la misma org y el org owner no quiere instalar la app multiples veces), entonces el aislamiento se implementa exclusivamente del lado de IAM con condition keys, aceptando que esto tiene la limitacion descrita en la seccion 2.

### Checklist de seguridad post-migracion

Cada uno de los siguientes puntos debe verificarse para completar la migracion de forma segura:

La GitHub App installation debe tener "Only select repositories" configurado (nunca "All repositories"). Cada service role de CodePipeline debe tener `codeconnections:UseConnection` con condition key `FullRepositoryId` restringido a sus repos. Los roles de CodeBuild que usen CodeConnections deben tener `GetConnectionToken` restringido al minimo necesario. Se debe habilitar CloudTrail para monitorear eventos de `GetConnectionToken` y `UseConnection`. Los branch protection rules en GitHub deben estar habilitados en ramas criticas. GitHub Secret Protection y Code Security deben habilitarse en los repos migrados. Los accesos HTTPS Git de CodeCommit deben revocarse una vez confirmada la migracion. El repositorio en CodeCommit debe marcarse como read-only o archivarse una vez completado el cutover.

## Automatizacion: Script de migracion por lotes

```bash
#!/bin/bash
# migrate-codecommit-to-github.sh
# Uso: ./migrate-codecommit-to-github.sh repos.txt MI-ORG us-east-1

REPO_LIST=$1
GITHUB_ORG=$2
AWS_REGION=$3

if [ -z "$REPO_LIST" ] || [ -z "$GITHUB_ORG" ] || [ -z "$AWS_REGION" ]; then
  echo "Uso: $0 <archivo-lista-repos> <github-org> <aws-region>"
  exit 1
fi

WORKDIR=$(mktemp -d)
REPORT="migration-report-$(date +%Y%m%d-%H%M%S).csv"
echo "repo,status,secrets_found,branches,tags" > "$REPORT"

while IFS= read -r REPO; do
  echo "=== Migrando: $REPO ==="

  # Clonar
  cd "$WORKDIR"
  git clone --mirror "https://git-codecommit.${AWS_REGION}.amazonaws.com/v1/repos/${REPO}" "${REPO}.git"
  if [ $? -ne 0 ]; then
    echo "$REPO,CLONE_FAILED,0,0,0" >> "$REPORT"
    continue
  fi

  cd "${REPO}.git"

  # Contar ramas y tags
  BRANCHES=$(git branch -a | wc -l)
  TAGS=$(git tag -l | wc -l)

  # Escanear secretos
  gitleaks detect --source . --report-format json --report-path "/tmp/${REPO}-secrets.json" 2>/dev/null
  SECRETS=$(cat "/tmp/${REPO}-secrets.json" 2>/dev/null | python3 -c "import sys,json; print(len(json.load(sys.stdin)))" 2>/dev/null || echo "0")

  if [ "$SECRETS" -gt 0 ]; then
    echo "$REPO,SECRETS_FOUND,$SECRETS,$BRANCHES,$TAGS" >> "$REPORT"
    echo "  ALERTA: $SECRETS secretos encontrados. Revisar /tmp/${REPO}-secrets.json"
    continue
  fi

  # Crear repo en GitHub
  gh repo create "${GITHUB_ORG}/${REPO}" --private --confirm 2>/dev/null

  # Push
  git remote add github "git@github.com:${GITHUB_ORG}/${REPO}.git"
  git push github --mirror

  if [ $? -eq 0 ]; then
    echo "$REPO,SUCCESS,0,$BRANCHES,$TAGS" >> "$REPORT"
  else
    echo "$REPO,PUSH_FAILED,0,$BRANCHES,$TAGS" >> "$REPORT"
  fi

  cd "$WORKDIR"
done < "$REPO_LIST"

echo ""
echo "=== Reporte: $REPORT ==="
cat "$REPORT"
```

El archivo `repos.txt` es simplemente una lista de nombres de repositorios de CodeCommit, uno por linea.

## Referencias

- [AWS CodePipeline: GitHub connections](https://docs.aws.amazon.com/codepipeline/latest/userguide/connections-github.html)
- [AWS CodeConnections: IAM permissions reference](https://docs.aws.amazon.com/dtconsole/latest/userguide/security-iam.html)
- [AWS CodeConnections: IAM policy examples](https://docs.aws.amazon.com/dtconsole/latest/userguide/security_iam_id-based-policy-examples-connections.html)
- [AWS CodeConnections: How connections work with GitHub organizations](https://docs.aws.amazon.com/dtconsole/latest/userguide/welcome-connections-how-it-works-github-organizations.html)
- [CodeStarSourceConnection action reference](https://docs.aws.amazon.com/codepipeline/latest/userguide/action-reference-CodestarConnectionSource.html)
- [Thomas Preece: Escalating Privileges via AWS CodeConnections (Part 1)](https://thomaspreece.com/2025/12/04/part-1-overview-of-aws-codeconnections-escalating-privileges-via-aws-codeconnections/)
- [Thomas Preece: AWS CodeBuild privilege escalation via CodeConnections (Part 2)](https://thomaspreece.com/2026/03/23/part-2-aws-codebuild-escalating-privileges-via-aws-codeconnections/)
- [AWS Service Authorization Reference: CodeConnections condition keys](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awscodeconnections.html)
