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

### Prerrequisitos (verificar antes de empezar)

- **AWS CLI con perfiles por cuenta** vía SSO (`mgmt`, `solutionhome`, `vault`, y uno por cuenta member);
  sin credenciales hardcodeadas (`aws configure sso`). Las sesiones SSO caducan — re-loguear con
  `aws sso login --profile <p>` cuando expiren.
- **Región primaria** definida (p. ej. `us-east-1`); usar la misma en la connection, el deploy y las validaciones.
- **GitHub — fork en una org/cuenta donde tengas rol de OWNER.** El handshake de CodeConnections requiere
  un **owner de la org de GitHub** para autorizar e instalar la app *AWS Connector for GitHub*. Si no eres
  owner de la org destino, escala a uno o usa una cuenta/org propia. (§A)
- **Cross-account backup opt-in** habilitado en el **management account** (§B). Sin esto, los `copy_actions`
  al central vault fallan con *"Cross-account backups are not opted-in"*.
- **AWS Config recorder por member:** si la cuenta ya tiene uno (p. ej. Control Tower
  `aws-controltower-BaselineConfigRecorder`) → `DeployConfigurationRecorder=no` (evita el error de recorder
  duplicado — AWS permite solo uno por cuenta/región); si no existe → `yes`. Verificar:
  `aws configservice describe-configuration-recorders --profile <member> --region <region>`.
- **Bucket de staging S3** en SolutionHome — el root template pesa >51.200 bytes y `cloudformation deploy`
  exige `--s3-bucket` (§D).

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
>
> **Requiere OWNER de la org de GitHub.** Completar el handshake (o crear la connection contra una
> instalación existente) exige un owner de la org con acceso a la instalación, y otorgar a la app permiso
> *Organization members: Read-only*. Si no eres owner, escala a uno o usa una cuenta/org propia.
> Validar: `aws codestar-connections get-connection --connection-arn <arn> --profile solutionhome --region <region>` → `ConnectionStatus: AVAILABLE`.

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

# Cross-account backup opt-in (org-wide) — IMPRESCINDIBLE para los copy_actions al central vault
aws backup update-global-settings \
  --global-settings isCrossAccountBackupEnabled=true \
  --profile mgmt --region us-east-1
```
**Validar:**
- `aws organizations list-delegated-administrators --profile mgmt` → aparece la SolutionHome (con
  `list-delegated-services-for-account` mostrando `backup.amazonaws.com` + `member.org.stacksets.cloudformation.amazonaws.com`).
- `aws backup describe-global-settings --profile mgmt --region us-east-1` → `"isCrossAccountBackupEnabled": "true"`
  (el cambio puede tardar unos minutos en propagar antes de que el primer copy job funcione).

---

## C. Cuentas SolutionHome y Vault Central

Si no existen, crearlas en la org (o reutilizar cuentas dedicadas). El Vault Central puede apuntar a una
cuenta dedicada (p. ej. Log Archive, ya restringida). Anotar:
- `<SOLUTIONHOME_ACCT_ID>`, OU de SolutionHome (`<SOLUTIONHOME_OU>`)
- `<CENTRAL_ACCT_ID>`, OU (`<CENTRAL_OU>`), región del central (`<CENTRAL_REGION>`)
- OUs objetivo de los member (`<TARGET_OUS>`), regiones a respaldar (`<TARGET_REGIONS>`)

---

## D. Desplegar el pipeline (root stack) — `--profile solutionhome`

El root template pesa >51.200 bytes → hay que stagearlo en S3. Crear el bucket una vez:
```
aws s3 mb s3://<staging-bucket> --profile solutionhome --region us-east-1
```

```
aws cloudformation deploy \
  --template-file _backup-recovery-with-aws-backup.yaml \
  --stack-name aws-backup-solution \
  --s3-bucket <staging-bucket> \
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
    CostCenter=<cc> Environment=<env> ApplicationOwner=<owner> Application=<app>
```
> Los parámetros obligatorios (sin default) son: `GitHubUsername`, `CodeConnectionArn`, `OrgId`,
> `OrgMgmtAcctId`, `SolutionHomeOrgUnit`, `TargetGlobalRegion`, `TargetRegions`, `TargetOUs`,
> `AWSBackupCentralAccountId/OU/Region`. El resto tiene default.
>
> **Nota `data-class`:** `ComplianceTagKey=data-class` / `ComplianceTagValue=regulated` — clave/valor del
> tag que selecciona el subconjunto regulado (tier de compliance annual).
>
> **Nota `DeployConfigurationRecorder`:** el ejemplo usa `yes`, pero si la cuenta member ya tiene un Config
> recorder (p. ej. Control Tower) hay que ponerlo en **`no`** (ver Prerrequisitos) — si no, el stackset falla.
>
> **Valores enum sensibles** (validados por CloudFormation en el deploy): `Environment` acepta
> `prod|stg|dev|sbx|qa` (alineado al tag policy de la organización). El template no define `BusinessUnit`
> (no forma parte del tag policy); no lo pases en `--parameter-overrides`. Las tag keys aplicadas son las
> del policy: `environment`, `application`, `cost-center`, `owner`.

---

## E. El pipeline despliega (automático) + gate del Vault ARN

Tras el stack, el CodePipeline corre: Source (GitHub) → Validate (cfn-nag) → Assets (S3) →
DeployCentralAcctResources (**crea el central vault**) → DeployBackupRecoveryMemberAccounts (rol + KMS+vault
+ Config framework) → **[GATE]** DeployBackupOrgPolicy → DeployReportPlan. Seguir en la Console o:
```
aws codepipeline get-pipeline-state --name backup-recovery-aws-backup --profile solutionhome --region us-east-1
```

**Gate del Vault ARN (`DeployBackupOrgPolicy`).** Este stage nace con la transición **deshabilitada**
(`DisableInboundStageTransitions`) como pausa de control. No requiere edición manual del template: la
**key** y el **value** del `copy_actions` se construyen automáticamente desde el ARN del central vault en
el SSM `/backup/central-vault-arn` vía `AWS::LanguageExtensions` (`Fn::ForEach`), en los cuatro planes
(daily/weekly/monthly/compliance). Único paso operativo: habilitar la transición:
```
aws codepipeline enable-stage-transition --pipeline-name backup-recovery-aws-backup \
  --stage-name DeployBackupOrgPolicy --transition-type Inbound --profile solutionhome --region us-east-1
```
> *Opcional:* al no requerir edición manual, la transición deshabilitada puede quitarse del root template
> para un despliegue sin pausas.

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

**Estructura desplegada:**
```
aws cloudformation list-stack-instances --stack-set-name <stackset> --call-as DELEGATED_ADMIN --profile solutionhome --region us-east-1
```

**Runtime — validación on-demand** (los backups programados corren en su cron; para probar ya, forzar un
job). Nota: `start-backup-job` **no** tiene `--copy-actions` → la copia cross-account se valida con un
`start-copy-job` aparte.
```
# 1. Taggear (o crear) un recurso backup-elegible en el member
aws ec2 create-volume --size 1 --volume-type gp3 --availability-zone <region>a \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=backup,Value=daily}]' --profile <member> --region <region>

# 2. Backup on-demand al member vault
aws backup start-backup-job \
  --backup-vault-name AWSBackupSolutionVault \
  --resource-arn arn:aws:ec2:<region>:<member-acct>:volume/<vol-id> \
  --iam-role-arn arn:aws:iam::<member-acct>:role/AWSBackupSolutionRole \
  --profile <member> --region <region>
# esperar COMPLETED: aws backup describe-backup-job --backup-job-id <id> ... (devuelve RecoveryPointArn)

# 3. Copy cross-account del recovery point al central vault (requiere el opt-in de §B)
aws backup start-copy-job \
  --recovery-point-arn <RecoveryPointArn> \
  --source-backup-vault-name AWSBackupSolutionVault \
  --destination-backup-vault-arn arn:aws:backup:<region>:<central-acct>:backup-vault:AWSBackupSolutionCentralVault \
  --iam-role-arn arn:aws:iam::<member-acct>:role/AWSBackupSolutionRole \
  --profile <member> --region <region>
# esperar COMPLETED: aws backup describe-copy-job --copy-job-id <id> ...

# Listados
aws backup list-backup-jobs --by-state COMPLETED --profile <member> --region <region>
aws backup list-copy-jobs   --by-state COMPLETED --profile <member> --region <region>
```
*Expected:* StackSets `SUCCEEDED`, backup job `COMPLETED` (recovery point en el member vault), copy job
`COMPLETED` (recovery point replicado en el central vault de la cuenta Vault).

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
- La solución no incluye recursos de demo ni de restore-testing.
- S3: SSE + PublicAccessBlock + versioning + logging (access-logs bucket dedicado por template).
- KMS/vault: `aws:PrincipalOrgID`/`PrincipalOrgPaths` (cross-account acotado a la org).
- Source vía **CodeConnections** (`CodeStarSourceConnection`), sin PAT en el pipeline.
- Post-deploy: least-privilege del pipeline role vía IAM Access Analyzer (paso H).

## Checklist de requisitos

Prerequisitos y puntos de atención del despliegue completo (backup + copy cross-account):

1. **Perfiles SSO** por cuenta; re-loguear al expirar (`aws sso login`).
2. **GitHub owner** de la org para el handshake de CodeConnections (§A).
3. **Cross-account backup opt-in** en management, `isCrossAccountBackupEnabled=true` (§B) — propaga en minutos.
4. **`DeployConfigurationRecorder=no`** si la cuenta member ya tiene Config recorder (Control Tower).
5. **`--s3-bucket`** en el deploy del root (template >51 KB) (§D).
6. **Enum case-sensitive**: `Environment` en minúsculas; el resto de toggles `yes`/`no` (entrecomillados en el YAML).
7. **CodeBuild image** `standard:7.0` (Ruby 3.x) para que `gem install cfn-nag` funcione.
8. **Gate del Vault ARN**: la key/value de `copy_actions` se construyen solas (LanguageExtensions/`Fn::ForEach` desde el SSM del ARN) — **sin edición manual**; solo habilitar la transición del stage (§E).
9. **Deploy role** con permisos S3 de bucket-config (ownership/logging/lifecycle) + backup report-plan (ListTags/ListReportPlans) — ya incluidos en el template.
