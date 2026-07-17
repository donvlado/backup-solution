# README-EXECUTE — Despliegue de la solución AWS Backup centralizado (CLI)

> Guía de despliegue **end-to-end por línea de comandos** de este artifact (fork customizado de
> `aws-samples/backup-recovery-with-aws-backup`). Cubre: registro en GitHub, creación de cuentas
> (SolutionHome, Vault Central), delegaciones desde management, despliegue del pipeline y validación.
> Reemplaza los placeholders `<...>` por los valores reales de tu organización.

## 0. Modelo y cuentas

El root template `_backup-recovery-with-aws-backup.yaml` se despliega en la cuenta **SolutionHome** y
crea un **CodePipeline** que: (1) trae el fork de GitHub, (2) valida templates (CodeBuild + cfn-nag),
(3) despliega vía **service-managed StackSets** a las OUs objetivo los stacks de member (rol, KMS+vault,
Config framework, report plan) y el stack del **central account**, y (4) crea las **AWS Organizations
backup policies** (modelo por cadencia: daily/weekly/monthly + compliance annual).

| Cuenta | Rol |
|--------|-----|
| **Management** | Solo una vez: trusted access + registro de delegated admins + resource policy de backup. |
| **SolutionHome** (delegada) | Hostea el pipeline; delegated admin de StackSets y de Backup Policies. |
| **Vault Central** | Almacén secundario (copias cross-account). El target puede ser una cuenta dedicada (p. ej. Log Archive). |
| **Member accounts** (por OU) | Reciben el rol + KMS+vault + Config framework vía StackSets. |

**Prerrequisitos:** AWS CLI configurada con perfiles por cuenta (`--profile mgmt|solutionhome|vault`);
sin credenciales hardcodeadas (SSO / `aws configure`). Fork del repo en GitHub. Region primaria definida.

### Método de despliegue: pipeline-driven, no CLI manual

Esta solución **no se despliega comando por comando**. El motor de despliegue es el **CI/CD nativo del
artifact (CodePipeline/CodeBuild)**: la línea de comandos solo interviene en el bootstrap (una vez) y en
tareas operativas. Los pasos de esta guía reflejan ese modelo — la Sección D lanza *un* stack que levanta
el pipeline, y las Secciones E–G son observar el pipeline, taggear y validar.

| Capa | Método | Quién lo ejecuta |
|------|--------|------------------|
| **Delegación en Management** (trusted access, delegated admin, resource policy) | CLI o Console — **una sola vez** | Operador |
| **Bootstrap del pipeline** (root stack `_backup-recovery-with-aws-backup.yaml`) | **Un solo** `cloudformation deploy` (o Console) | Operador |
| **Despliegue real** (StackSets member/central + Config framework + report plan + las 4 org backup policies) | **Automático — CodePipeline/CodeBuild** | El pipeline |
| **Operativo** (taggear recursos, inyectar el vault ARN en el stage gateado, validar jobs) | CLI puntual | Operador |

> El pipeline nativo del artifact es el mecanismo de despliegue repetible; el hardening de seguridad vive
> en el código del artifact (S3, KMS, roles, source → CodeConnections), no en scripts manuales.

---

## A. GitHub — fork y registro

```
# 1. Fork/clonar este repo en tu GitHub y pushear el contenido customizado
git init && git add . && git commit -m "backup solution - initial customization"
git remote add origin https://github.com/<tu-org>/<repo>.git
git push -u origin main
```
> El pipeline usa **AWS CodeConnections** como source (provider `CodeStarSourceConnection`) — **sin PAT**.
> Antes del deploy, crear la connection en la cuenta SolutionHome y completar el handshake OAuth:
> ```
> aws codestar-connections create-connection --provider-type GitHub \
>   --connection-name <nombre> --profile solutionhome --region <REGION_PRIMARIA>
> ```
> Devuelve un `ConnectionArn` en estado `PENDING`. Autorizarlo en Console (Developer Tools → Settings →
> Connections → Update pending connection → instalar la app *AWS Connector for GitHub* sobre el repo del
> fork) hasta `AVAILABLE`. El ARN va al parámetro `CodeConnectionArn` (paso D).

---

## B. Management — habilitaciones y delegación (una sola vez, `--profile mgmt`)

```
# Trusted access: StackSets + AWS Backup
aws organizations enable-aws-service-access --service-principal member.org.stacksets.cloudformation.amazonaws.com
aws organizations enable-aws-service-access --service-principal backup.amazonaws.com

# Delegated administrator de la SolutionHome (StackSets + Backup)
aws organizations register-delegated-administrator \
  --account-id <SOLUTIONHOME_ACCT_ID> --service-principal member.org.stacksets.cloudformation.amazonaws.com
aws organizations register-delegated-administrator \
  --account-id <SOLUTIONHOME_ACCT_ID> --service-principal backup.amazonaws.com

# Resource policy para delegar la administración de Backup Policies a la SolutionHome.
# El archivo es un template CloudFormation (AWS::Organizations::ResourcePolicy) → se despliega con cloudformation deploy
# (NO con put-resource-policy, que espera un JSON de policy, no un template).
aws cloudformation deploy \
  --template-file cloudformation/stacks/aws-backup-org-resource-policy-delegate-backup-policy-mgmt.yaml \
  --stack-name aws-backup-org-delegate-backup-policy \
  --parameter-overrides \
    SolutionHomeAccountId=<SOLUTIONHOME_ACCT_ID> pOrganizationId=<o-xxxx> pOrganizationRootId=<r-xxxx> \
  --profile mgmt --region us-east-1
```
**Validar:** `aws organizations list-delegated-administrators` → aparece la SolutionHome.

---

## C. Cuentas SolutionHome y Vault Central

Si no existen, crearlas en la org (o reutilizar cuentas dedicadas). El Vault Central puede apuntar a una
cuenta dedicada (p. ej. Log Archive, ya restringida). Anotar:
- `<SOLUTIONHOME_ACCT_ID>`, OU de SolutionHome (`<SOLUTIONHOME_OU>`)
- `<CENTRAL_ACCT_ID>`, OU (`<CENTRAL_OU>`), región del central (`<CENTRAL_REGION>`)
- OUs objetivo de los member (`<TARGET_OUS>`), regiones a respaldar (`<TARGET_REGIONS>`)

---

## D. Desplegar el pipeline (root stack) — `--profile solutionhome`

```
aws cloudformation deploy \
  --template-file _backup-recovery-with-aws-backup.yaml \
  --stack-name aws-backup-solution \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
    GitHubRepo=<repo> GitHubBranch=main GitHubUsername=<tu-org> \
    CodeConnectionArn=<CONNECTION_ARN> \
    OrgId=<o-xxxx> OrgMgmtAcctId=<MGMT_ACCT_ID> SolutionHomeOrgUnit=<SOLUTIONHOME_OU> \
    TargetGlobalRegion=<region> TargetRegions=<TARGET_REGIONS> TargetOUs=<TARGET_OUS> \
    AWSBackupCentralAccountOU=<CENTRAL_OU> AWSBackupCentralAccountId=<CENTRAL_ACCT_ID> \
    AWSBackupCentralAccountRegion=<CENTRAL_REGION> \
    DailyBackupSchedule="cron(0 5 ? * * *)" WeeklyBackupSchedule="cron(0 5 ? * 1 *)" \
    MonthlyBackupSchedule="cron(0 5 1 * ? *)" AnnualBackupSchedule="cron(0 5 1 1 ? *)" \
    DailyRetentionDays=35 WeeklyRetentionDays=90 MonthlyRetentionDays=365 AnnualRetentionDays=2555 \
    ColdStorageAfterDays=30 \
    OperationalTagKey=backup OperationalTagValue=daily \
    ComplianceTagKey=data-class ComplianceTagValue=regulated \
    DeployConfigurationRecorder=yes DeployReportFeature=yes \
    BusinessUnit=<bu> CostCenter=<cc> Environment=<env> ApplicationOwner=<owner> Application=<app>
```
> Los parámetros obligatorios (sin default) son: `GitHubUsername`, `CodeConnectionArn`, `OrgId`,
> `OrgMgmtAcctId`, `SolutionHomeOrgUnit`, `TargetGlobalRegion`, `TargetRegions`, `TargetOUs`,
> `AWSBackupCentralAccountId/OU/Region`. El resto tiene default.
>
> **Nota `data-class`:** `ComplianceTagKey=data-class` / `ComplianceTagValue=regulated` — clave/valor del
> tag que selecciona el subconjunto regulado (tier de compliance annual).

---

## E. El pipeline despliega (automático)

Tras el stack, el CodePipeline corre: Source (GitHub) → Validate (cfn-nag) → StackSets (member role,
member KMS+vault, central account, Config framework, report plan) → org backup policies. Seguir en la
Console de CodePipeline o:
```
aws codepipeline get-pipeline-state --name <pipeline-name> --profile solutionhome
```

---

## F. Taggear recursos (en las cuentas member)

```
# Cadencia operativa
aws resourcegroupstaggingapi tag-resources --resource-arn-list <arn> --tags backup=daily
# Subconjunto regulado (overlay annual 7 años)
aws resourcegroupstaggingapi tag-resources --resource-arn-list <arn> --tags data-class=regulated
```
Valores de `backup`: `daily|weekly|monthly|none` (`none` excluye explícitamente).

---

## G. Validar

```
# StackSets en estado correcto
aws cloudformation list-stack-instances --stack-set-name <stackset> --call-as DELEGATED_ADMIN --profile solutionhome
# Backup jobs
aws backup list-backup-jobs --by-state COMPLETED --profile <member>
# Copy jobs cross-account al central vault
aws backup list-copy-jobs --by-state COMPLETED --profile <member>
```
*Expected:* StackSets `SUCCEEDED`, backup job `COMPLETED`, copy job cross-account `COMPLETED`.

---

## H. Post-deploy — least-privilege de roles

Con el pipeline ya ejecutado (historial en CloudTrail), generar la policy de mínimos privilegios del
`CodePipelineManageStepsRole` con **IAM Access Analyzer**:
```
aws accessanalyzer start-policy-generation \
  --policy-generation-details '{"principalArn":"<role-arn-CodePipelineManageStepsRole>"}' \
  --cloud-trail-details '{"trails":[{"cloudTrailArn":"<trail-arn>","allRegions":true}],"startTime":"<inicio>","endTime":"<fin>"}'
aws accessanalyzer get-generated-policy --job-id <job-id>
```
Reemplazar los statements con `Resource:"*"` por la policy generada, y complementar con un analyzer de
**Unused Access** para recortar lo no usado.

---

## Notas de seguridad
- Demo y restore-testing **eliminados** del artifact (no se despliegan).
- S3: SSE + PublicAccessBlock + versioning + logging (access-logs bucket dedicado por template).
- KMS/vault: `aws:PrincipalOrgID`/`PrincipalOrgPaths` (cross-account acotado a la org).
- Source vía **CodeConnections** (`CodeStarSourceConnection`), sin PAT en el pipeline.
- Post-deploy: least-privilege del pipeline role vía IAM Access Analyzer (paso H).
