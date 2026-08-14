# 海丝客户档案系统实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在固定版本 NocoBase 社区版上交付可自托管、可审计、可备份的内部客户健康档案系统。

**Architecture:** 仓库中的 `app/` 是由 `create-nocobase-app@2.1.39` 生成的 Yarn Workspace，业务能力集中在 `@haisi/plugin-customer-archive` 插件。插件以服务端集合和自定义事务操作作为唯一业务入口，前端使用稳定版 NocoBase `SchemaComponent`、React 与 Ant Design 组合内置表格和表单；PostgreSQL、附件和备份通过 Docker Compose 持久化。

**Tech Stack:** TypeScript, NocoBase 2.1.39, React, Ant Design, Node.js 22, Koa, PostgreSQL 16, Vitest, Playwright, Docker Compose, Nginx

**Spec:** `docs/superpowers/specs/2026-08-14-customer-archive-design.md`

## Global Constraints

- 固定 NocoBase Community Edition `2.1.39`，不得使用浮动 `latest` 标签。
- 使用 Node.js 22 与 Yarn 1.22.x；不得依赖当前主机的 Node.js 25 行为。
- 插件包名固定为 `@haisi/plugin-customer-archive`。
- 不修改 NocoBase 核心源码，不引入维持第一版运行所必需的商业插件。
- 生产业务界面使用稳定版 `src/client/` API；`src/client-v2/` 只保留脚手架入口，不承载第一版核心流程。
- PostgreSQL 保存 UTC 时间，界面和定时任务使用 `Asia/Shanghai`。
- 手机号可空、可重复，不执行重复查询或警告。
- 客户创建与授权确认必须在同一事务内完成；标准 `customers:create` 不向普通业务角色开放。
- `tmp/`、真实体检报告、`.env*`、附件、备份和对象存储凭证不得提交到 Git。
- 健康附件仅允许 PDF、JPG、JPEG、PNG，单文件上限 30 MB。
- 首批正式账号全部为管理员，但发布前必须用测试普通员工验证数据隔离。
- 所有写操作必须有自动测试；每个任务完成后先运行对应测试再提交。

## File Map

- `app/`: 固定版本 NocoBase 应用工作区。
- `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/`: 业务集合定义。
- `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/`: 客户、健康、授权、导入导出和备份业务逻辑。
- `app/packages/plugins/@haisi/plugin-customer-archive/src/server/actions/`: NocoBase ResourceManager 操作处理器。
- `app/packages/plugins/@haisi/plugin-customer-archive/src/server/policies/`: 管理员与普通员工数据范围判断。
- `app/packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/`: 服务端单元和集成测试。
- `app/packages/plugins/@haisi/plugin-customer-archive/src/client/`: 稳定版客户端入口、页面与可复用表单。
- `app/packages/plugins/@haisi/plugin-customer-archive/src/locale/`: 中英文文案。
- `deploy/`: Docker Compose、Dockerfile、Nginx 和运维脚本。
- `e2e/`: Playwright 桌面与手机端验收测试。
- `docs/operations/`: 安装、备份、恢复、升级和云迁移说明。

---

### Task 1: 固定版本应用底座与仓库基线

**Files:**
- Create: `app/package.json` and generated NocoBase workspace files
- Create: `app/.nvmrc`
- Create: `app/.env.example`
- Create: `deploy/compose.dev.yml`
- Modify: `.gitignore`
- Test: `scripts/check-scaffold.sh`

**Interfaces:**
- Produces: NocoBase 2.1.39 workspace at `app/`, PostgreSQL service name `postgres`, application port `13000`.

- [ ] **Step 1: Write the scaffold verification script**

```sh
#!/bin/sh
set -eu
test "$(node -p "require('./app/package.json').dependencies['@nocobase/app']")" = "2.1.39"
test -d app/packages/plugins
docker compose -f deploy/compose.dev.yml config >/dev/null
```

- [ ] **Step 2: Run it and verify it fails before scaffolding**

Run: `sh scripts/check-scaffold.sh`

Expected: non-zero exit because `app/package.json` does not exist.

- [ ] **Step 3: Generate the pinned NocoBase application and plugin skeleton**

Run:

```bash
npx create-nocobase-app@2.1.39 app -d postgres \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  -e DB_DATABASE=haisi \
  -e DB_USER=haisi \
  -e DB_PASSWORD=haisi_dev_only \
  -e TZ=UTC
cd app
npx yarn@1.22.22 pm create @haisi/plugin-customer-archive
```

Set `app/.nvmrc` to `22`, add `APPEND_PRESET_BUILT_IN_PLUGINS=@haisi/plugin-customer-archive` to `.env.example`, and create a PostgreSQL 16 development service with named volume `haisi_pg_dev` in `deploy/compose.dev.yml`.

- [ ] **Step 4: Install dependencies and verify the workspace**

Run:

```bash
cd app
npx yarn@1.22.22 install --frozen-lockfile
npx yarn@1.22.22 nocobase --version
```

Expected: version output contains `2.1.39`.

- [ ] **Step 5: Run the scaffold verification and commit**

Run: `sh scripts/check-scaffold.sh`

Expected: PASS.

```bash
git add .gitignore app deploy/compose.dev.yml scripts/check-scaffold.sh
git commit -m "chore: scaffold pinned NocoBase application"
```

---

### Task 2: 业务集合与初始数据

**Files:**
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/customers.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/healthRecords.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/sourceTypes.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/customerTags.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/consentTemplates.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/consentConfirmations.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/customerNumberCounters.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/backupSettings.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/backupRuns.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/seeds.ts`
- Test: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/collections.test.ts`

**Interfaces:**
- Produces: collections named `customers`, `healthRecords`, `sourceTypes`, `customerTags`, `consentTemplates`, `consentConfirmations`, `customerNumberCounters`, `backupSettings`, `backupRuns`.

- [ ] **Step 1: Write failing collection tests**

```ts
it('allows duplicate and empty mobile values', async () => {
  await customerRepo.create({ values: { customerNo: 'HS-202608-000001', name: '甲', ownerId: 1 } });
  await customerRepo.create({ values: { customerNo: 'HS-202608-000002', name: '乙', mobile: '13800000000', ownerId: 1 } });
  await expect(customerRepo.create({
    values: { customerNo: 'HS-202608-000003', name: '丙', mobile: '13800000000', ownerId: 1 },
  })).resolves.toBeDefined();
});

it('keeps many health records for one customer', async () => {
  await healthRepo.createMany({ records: [
    { customerId, recordedAt: '2026-08-01' },
    { customerId, recordedAt: '2026-08-14' },
  ] });
  expect(await healthRepo.count({ filter: { customerId } })).toBe(2);
});
```

- [ ] **Step 2: Verify the tests fail because collections are absent**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/collections.test.ts`

Expected: FAIL with collection not found.

- [ ] **Step 3: Define collections using `defineCollection()`**

Use one collection per file. The customer mobile field must be nullable and non-unique:

```ts
{ name: 'mobile', type: 'string', allowNull: true, unique: false, interface: 'phone' }
```

Define `healthRecords.attachments` as a many-to-many attachment association targeting `attachments`; define customer `tags` through `customersTags`; define `owner`, `createdBy`, and association foreign keys explicitly. Add indexes to `customerNo`, `name`, `mobile`, `ownerId`, `status`, `recordedAt`, and `customerId`.

- [ ] **Step 4: Seed idempotent source types, consent version 1.0, and backup defaults**

```ts
export const defaultSourceTypes = [
  ['walk_in', '自然到店'],
  ['customer_referral', '老客户介绍'],
  ['employee_referral', '员工介绍'],
  ['wechat', '微信'],
  ['event', '活动'],
  ['other', '其他'],
] as const;
```

`seedInitialData(db)` must use `updateOrCreate` keyed by stable code/version, set `localEnabled=true`, `cloudEnabled=false`, `preUpgradeBackupEnabled=true`, and never overwrite an administrator-edited current consent body after installation.

- [ ] **Step 5: Run collection tests and commit**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/collections.test.ts`

Expected: PASS.

```bash
git add app/packages/plugins/@haisi/plugin-customer-archive
git commit -m "feat: define customer archive collections"
```

---

### Task 3: 原子建档、客户编号与授权生命周期

**Files:**
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/customer-number.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/customer-service.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/actions/customers.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/validators/customer.ts`
- Test: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/customer-service.test.ts`

**Interfaces:**
- Produces: `CustomerService.createWithConsent(input, actorId): Promise<CustomerModel>`.
- Produces: actions `customers:createWithConsent`, `customers:archive`, `customers:restore`, `consentConfirmations:revoke`.

- [ ] **Step 1: Write failing transaction and numbering tests**

```ts
it('creates customer and immutable consent in one transaction', async () => {
  const customer = await service.createWithConsent({ name: '张三', consentConfirmed: true }, adminId);
  expect(customer.get('customerNo')).toMatch(/^HS-\d{6}-\d{6}$/);
  const consent = await consentRepo.findOne({ filter: { customerId: customer.get('id') } });
  expect(consent.get('templateVersion')).toBe('1.0');
  expect(consent.get('contentSnapshot')).toContain('海丝企业服务中心');
});

it('rolls back customer when consent creation fails', async () => {
  vi.spyOn(consentRepo, 'create').mockRejectedValueOnce(new Error('forced failure'));
  await expect(service.createWithConsent({ name: '李四', consentConfirmed: true }, adminId)).rejects.toThrow();
  expect(await customerRepo.count({ filter: { name: '李四' } })).toBe(0);
});
```

- [ ] **Step 2: Verify tests fail**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/customer-service.test.ts`

Expected: FAIL because `CustomerService` is absent.

- [ ] **Step 3: Implement the monthly sequence under a database transaction**

```ts
const period = formatInTimeZone(now, 'Asia/Shanghai', 'yyyyMM');
const counter = await Counter.findOne({ where: { period }, transaction, lock: transaction.LOCK.UPDATE });
const nextValue = counter ? counter.nextValue + 1 : 1;
await Counter.upsert({ period, nextValue }, { transaction });
return `HS-${period}-${String(nextValue).padStart(6, '0')}`;
```

Enforce a unique database constraint on `customerNo`; retry once on a unique-conflict caused by concurrent creation.

- [ ] **Step 4: Implement atomic customer and consent creation**

Reject missing `name`, false `consentConfirmed`, or missing current consent template. Inside one `db.sequelize.transaction`, generate the number, create customer with `ownerId ?? actorId`, and create the consent snapshot with `confirmedById=actorId` and server time. Strip `consentConfirmed` before persisting the customer.

- [ ] **Step 5: Implement archive, restore, and revoke rules**

Archive and restore change only `customers.status`. Revoke sets `revokedAt`, `revokedById`, and required `revocationReason`; it never edits the snapshot. A customer with a revoked latest consent cannot receive a new health record until a new consent confirmation is created.

- [ ] **Step 6: Register custom actions and deny standard business creation**

```ts
this.app.resourceManager.registerActionHandlers({
  'customers:createWithConsent': createWithConsentAction(this.customerService),
  'customers:archive': archiveAction(this.customerService),
  'customers:restore': restoreAction(this.customerService),
  'consentConfirmations:revoke': revokeConsentAction(this.customerService),
});
```

- [ ] **Step 7: Run tests and commit**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/customer-service.test.ts`

Expected: PASS, including concurrent numbering test.

```bash
git add app/packages/plugins/@haisi/plugin-customer-archive
git commit -m "feat: create customers with audited consent"
```

---

### Task 4: 健康记录与附件校验

**Files:**
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/validators/health-record.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/health-record-service.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/actions/health-records.ts`
- Test: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/health-record-service.test.ts`

**Interfaces:**
- Produces: `HealthRecordService.create(input, actor): Promise<HealthRecordModel>`.
- Produces: `healthRecords:createValidated` and `healthRecords:updateValidated` actions.

- [ ] **Step 1: Write failing boundary tests**

```ts
it.each([
  ['heightCm', 29], ['heightCm', 251], ['weightKg', 0], ['systolic', 301],
  ['diastolic', 201], ['heartRate', 301],
])('rejects invalid %s=%s', async (field, value) => {
  await expect(service.create({ customerId, recordedAt: '2026-08-14', [field]: value }, actor))
    .rejects.toMatchObject({ status: 422 });
});
```

Also test PDF/JPEG/PNG acceptance, executable rejection, 30 MB boundary, multiple records per customer, and revoked-consent rejection.

- [ ] **Step 2: Verify tests fail**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/health-record-service.test.ts`

Expected: FAIL because validator and service are absent.

- [ ] **Step 3: Implement validation and service methods**

Use a declarative range map:

```ts
const ranges = {
  heightCm: [30, 250], weightKg: [1, 500], systolic: [40, 300],
  diastolic: [20, 200], heartRate: [20, 300],
} as const;
```

Validate attachment MIME type and byte size on the server; do not trust the filename extension. Ensure the customer exists, is visible to the actor, is not archived, and has a non-revoked latest consent.

- [ ] **Step 4: Register validated actions and tests**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/health-record-service.test.ts`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/packages/plugins/@haisi/plugin-customer-archive
git commit -m "feat: validate health records and attachments"
```

---

### Task 5: 权限、审计与安全边界

**Files:**
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/policies/customer-scope.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/middleware/audit.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/collections/customerAuditLogs.ts`
- Modify: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/plugin.ts`
- Test: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/permissions.test.ts`

**Interfaces:**
- Produces: `canAccessCustomer(actor, customer): boolean` and `customerScopeFilter(actor)`.
- Produces: append-only `customerAuditLogs` records with actor, action, collection, record ID, result, and timestamp.

- [ ] **Step 1: Write failing administrator and staff isolation tests**

```ts
expect(await listAs(admin)).toHaveLength(2);
expect(await listAs(staffA)).toEqual(expect.arrayContaining([
  expect.objectContaining({ ownerId: staffA.id }),
]));
expect(await getAs(staffA, customerOwnedByB.id)).toMatchObject({ status: 403 });
expect(await destroyAs(staffA, customerOwnedByA.id)).toMatchObject({ status: 403 });
```

- [ ] **Step 2: Verify tests fail with over-broad access**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/permissions.test.ts`

Expected: at least the cross-owner assertion fails.

- [ ] **Step 3: Implement the exact staff filter**

```ts
return isAdmin(actor) ? {} : {
  $or: [
    { ownerId: actor.id },
    { createdById: actor.id },
  ],
};
```

Apply the same customer scope transitively to health records and attachments. Only administrators receive management, audit, import/export, archive/restore, delete, and backup actions.

- [ ] **Step 4: Add append-only audit middleware**

Audit successful and failed writes without serializing health text, consent body, credentials, tokens, or attachment bytes. Block `update` and `destroy` for `consentConfirmations` except the registered revoke action.

- [ ] **Step 5: Run tests and commit**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/permissions.test.ts`

Expected: PASS.

```bash
git add app/packages/plugins/@haisi/plugin-customer-archive
git commit -m "feat: enforce customer scope and audit writes"
```

---

### Task 6: 品牌、内置页面和手机布局

**Files:**
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/collections.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/schemas/customer-list.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/schemas/customer-detail.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/pages/CustomerCreatePage.tsx`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/pages/CustomerListPage.tsx`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/pages/CustomerDetailPage.tsx`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/pages/ManagementPage.tsx`
- Modify: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/plugin.tsx`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/locale/zh-CN.json`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/locale/en-US.json`
- Test: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/__tests__/customer-create.test.tsx`

**Interfaces:**
- Produces routes `/admin/haisi/customers`, `/admin/haisi/customers/new`, `/admin/haisi/customers/:id`, `/admin/haisi/settings`.
- Consumes `customers:createWithConsent` and validated health actions.

- [ ] **Step 1: Write failing client tests**

```tsx
render(<CustomerCreatePage />);
expect(screen.getByText('客户信息与健康资料授权告知')).toBeInTheDocument();
expect(screen.getByRole('button', { name: '保存' })).toBeDisabled();
await user.click(screen.getByRole('checkbox'));
expect(screen.getByRole('button', { name: '保存' })).toBeEnabled();
```

Add tests that consent status is absent from the list/basic-information DOM, phone is not required, and mobile width renders a one-column form.

- [ ] **Step 2: Verify tests fail**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/client/__tests__/customer-create.test.tsx`

Expected: FAIL because pages are absent.

- [ ] **Step 3: Register collection metadata and stable client routes**

Use `this.app.getCollectionManager().addCollections(customerArchiveCollections)` in the stable client entry. Register routes in `load()` and render NocoBase `SchemaComponent` for generic list/detail blocks. Keep only consent submission and backup settings as small React components.

- [ ] **Step 4: Implement desktop and mobile schemas**

The desktop customer table includes number, name, sex, calculated age, mobile, source, referrer, tags, owner, status, and established date. The compact mobile card includes number, name, mobile, source, owner, and status. Detail tabs are 基本信息、健康记录、操作记录; the third tab is administrator-only.

- [ ] **Step 5: Apply brand assets without hiding required NocoBase notices**

Copy `logo01.jpg` into the plugin assets, set the application title to `海丝企业服务中心客户管理系统`, disable self-registration, and retain upstream platform attribution outside the replaceable main Logo.

- [ ] **Step 6: Run tests, build, and commit**

Run:

```bash
cd app
npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/client
npx yarn@1.22.22 build @haisi/plugin-customer-archive
```

Expected: tests PASS and plugin build exits 0.

```bash
git add app/packages/plugins/@haisi/plugin-customer-archive
git commit -m "feat: add branded customer management pages"
```

---

### Task 7: 客户 Excel 导入与导出

**Files:**
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/customer-xlsx.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/actions/customer-xlsx.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/components/CustomerXlsxActions.tsx`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/assets/customer-import-template.xlsx`
- Test: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/customer-xlsx.test.ts`

**Interfaces:**
- Produces: `customers:importXlsx` and `customers:exportXlsx` administrator actions.
- Consumes: `CustomerService.createWithConsent()` for every imported row.

- [ ] **Step 1: Write failing workbook tests**

```ts
expect(parseRow({ 姓名: '王五', 手机号: '', 授权已确认: '是' })).toMatchObject({
  name: '王五', mobile: undefined, consentConfirmed: true,
});
expect(() => parseRow({ 姓名: '赵六', 授权已确认: '否' })).toThrow('授权已确认必须为“是”');
```

Test duplicate phones, Chinese text, valid-row import, invalid-row rollback, error line numbers, and admin-only export.

- [ ] **Step 2: Verify tests fail**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/customer-xlsx.test.ts`

Expected: FAIL because XLSX service is absent.

- [ ] **Step 3: Add and pin ExcelJS**

Run: `cd app && npx yarn@1.22.22 workspace @haisi/plugin-customer-archive add exceljs@4.4.0`

- [ ] **Step 4: Implement row-atomic import and permission-aware export**

Map the fixed Chinese columns to service input. Each row calls `createWithConsent` in its own transaction; collect `{ row, message }` errors without leaving half a row. Export only records allowed by `customerScopeFilter(actor)`; health text and attachment binaries are excluded.

- [ ] **Step 5: Run tests and commit**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/customer-xlsx.test.ts`

Expected: PASS.

```bash
git add app/packages/plugins/@haisi/plugin-customer-archive app/yarn.lock
git commit -m "feat: import and export customer workbooks"
```

---

### Task 8: 本地与可选云端备份

**Files:**
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/backup/backup-service.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/backup/local-backup.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/backup/cloud-provider.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/backup/tencent-cos.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/backup/huawei-obs.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/services/backup/credential-vault.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/actions/backups.ts`
- Create: `app/packages/plugins/@haisi/plugin-customer-archive/src/client/pages/BackupSettingsPage.tsx`
- Test: `app/packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/backup-service.test.ts`

**Interfaces:**
- Produces: `BackupService.run(reason)`, `BackupService.updateSettings(input, actor)`, adapters implementing `upload(localPath, remoteKey)`.
- Produces actions `backups:settings`, `backups:updateSettings`, `backups:run`, `backups:listRuns`.

- [ ] **Step 1: Write failing switch and retention tests**

```ts
it('does not call cloud provider while cloud backup is disabled', async () => {
  await service.run('scheduled');
  expect(cloud.upload).not.toHaveBeenCalled();
});

it('stops future uploads without deleting existing cloud objects', async () => {
  await service.updateSettings({ cloudEnabled: false }, admin);
  expect(cloud.remove).not.toHaveBeenCalled();
});
```

Also test matched database/attachment timestamp, 7 daily plus 4 weekly retention, masked credentials, authentication-tag failure, manual run audit, and non-admin rejection.

- [ ] **Step 2: Verify tests fail**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/backup-service.test.ts`

Expected: FAIL because backup services are absent.

- [ ] **Step 3: Implement safe local backup execution**

Use `execFile`, never a shell-built command string:

```ts
await execFileAsync('pg_dump', ['--format=custom', '--file', dbPath, databaseUrl], {
  env: sanitizedEnv,
});
await execFileAsync('tar', ['-czf', filesPath, '-C', uploadsRoot, '.']);
```

Write a JSON manifest containing app version, plugin version, timestamp, database checksum, attachment checksum, and reason. Mark the run successful only after all files and checksums exist.

- [ ] **Step 4: Implement encrypted credentials and cloud adapters**

Use AES-256-GCM with a 32-byte `BACKUP_MASTER_KEY` supplied only through environment variables. Persist `{ iv, tag, ciphertext }`; API responses return `******` rather than ciphertext or plaintext. Add `cos-nodejs-sdk-v5@3.0.0` (ISC) and `esdk-obs-nodejs@3.26.2` (Apache-2.0) as pinned plugin dependencies; adapters upload the database dump, attachment archive, and manifest under the same remote prefix.

- [ ] **Step 5: Register the Asia/Shanghai daily job and settings UI**

Use `this.app.cronJobManager.addJob` with `cronTime` from settings, `timeZone: 'Asia/Shanghai'`, and a guard that exits when `localEnabled=false`. Cloud upload runs only when `cloudEnabled=true`. The UI displays disabled-risk warning, last run, result, size, provider, and masked credential fields.

- [ ] **Step 6: Run tests and commit**

Run: `cd app && npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src/server/__tests__/backup-service.test.ts`

Expected: PASS.

```bash
git add app/packages/plugins/@haisi/plugin-customer-archive app/yarn.lock
git commit -m "feat: add optional local and cloud backups"
```

---

### Task 9: Docker、Nginx、恢复和升级保护

**Files:**
- Create: `deploy/Dockerfile`
- Create: `deploy/compose.prod.yml`
- Create: `deploy/nginx/haisi.conf`
- Create: `scripts/backup-now.sh`
- Create: `scripts/restore.sh`
- Create: `scripts/upgrade.sh`
- Create: `docs/operations/installation.md`
- Create: `docs/operations/backup-restore.md`
- Create: `docs/operations/upgrade.md`
- Test: `scripts/check-deployment.sh`

**Interfaces:**
- Produces: production services `app`, `postgres`, `nginx`; volumes `haisi_pg_data`, `haisi_uploads`, `haisi_backups`.
- Consumes: `PRE_UPGRADE_BACKUP_ENABLED` and explicit `--acknowledge-no-backup` override.

- [ ] **Step 1: Write failing deployment checks**

```sh
#!/bin/sh
set -eu
docker compose -f deploy/compose.prod.yml config >/dev/null
grep -q 'client_max_body_size 30m' deploy/nginx/haisi.conf
grep -q 'proxy_set_header X-Forwarded-Proto' deploy/nginx/haisi.conf
grep -q -- '--acknowledge-no-backup' scripts/upgrade.sh
```

- [ ] **Step 2: Verify checks fail**

Run: `sh scripts/check-deployment.sh`

Expected: non-zero exit because deployment files are absent.

- [ ] **Step 3: Build the pinned production image**

Use `node:22-bookworm-slim`, install PostgreSQL client and CA certificates, run Yarn 1.22.22 with the committed lockfile, build `@haisi/plugin-customer-archive`, and run NocoBase as a non-root user. The image must expose 13000 and include an HTTP health check.

- [ ] **Step 4: Implement safe backup, restore, and upgrade scripts**

`restore.sh` requires explicit database dump and attachment archive paths, verifies both checksums from the manifest, restores into a named non-production target unless `--production` is explicitly supplied, and never accepts `/`, `~`, or an unresolved environment variable as a destination.

`upgrade.sh` creates a backup by default. If disabled, it exits unless the administrator passes `--acknowledge-no-backup`; the script records old version, new version, operator, timestamp, and acknowledgement in `backups/upgrade-audit.log`.

- [ ] **Step 5: Validate configurations and build**

Run:

```bash
sh scripts/check-deployment.sh
docker compose -f deploy/compose.prod.yml build app
docker compose -f deploy/compose.prod.yml config
```

Expected: all commands exit 0.

- [ ] **Step 6: Commit**

```bash
git add deploy scripts docs/operations
git commit -m "ops: add reproducible deployment and recovery"
```

---

### Task 10: 端到端验收、恢复演练和发布

**Files:**
- Create: `e2e/customer-admin.spec.ts`
- Create: `e2e/customer-staff.spec.ts`
- Create: `e2e/mobile-customer.spec.ts`
- Create: `e2e/backup-settings.spec.ts`
- Create: `e2e/fixtures/report-sample.pdf`
- Modify: `README.md`
- Create: `docs/operations/acceptance-report.md`

**Interfaces:**
- Consumes all prior application, permission, upload, import/export, and backup interfaces.
- Produces a repeatable release verification command and evidence report.

- [ ] **Step 1: Write the failing administrator flow**

```ts
test('administrator creates an audited customer and health record', async ({ page }) => {
  await loginAsAdmin(page);
  await page.getByRole('link', { name: '客户管理' }).click();
  await page.getByRole('button', { name: '新建客户' }).click();
  await page.getByLabel('姓名').fill('端到端测试客户');
  await page.getByRole('checkbox', { name: /已向客户完整说明/ }).check();
  await page.getByRole('button', { name: '保存' }).click();
  await expect(page.getByText(/^HS-\d{6}-\d{6}$/)).toBeVisible();
});
```

- [ ] **Step 2: Add staff, mobile, import/export, upload and backup flows**

The staff test must receive 403 for another owner's customer. The mobile test uses a 390×844 viewport, creates and edits a customer, adds a health record, and uploads the synthetic fixture. The backup test verifies the default cloud-off state, warning behavior, manual local backup, and masked credentials. The fixture must be generated synthetic data and contain no real person or health information.

- [ ] **Step 3: Run all automated verification**

Run:

```bash
cd app
npx yarn@1.22.22 test packages/plugins/@haisi/plugin-customer-archive/src
npx yarn@1.22.22 build @haisi/plugin-customer-archive
cd ..
npx playwright test e2e
sh scripts/check-deployment.sh
```

Expected: all tests PASS and build exits 0.

- [ ] **Step 4: Perform a clean restore drill**

Create a backup from the test environment, restore it into a fresh PostgreSQL database and empty uploads volume, start the restored app, then rerun the administrator read-only assertions. Record customer, health, consent and attachment counts plus checksum results in `docs/operations/acceptance-report.md`.

- [ ] **Step 5: Review repository for secrets and real health data**

Run:

```bash
git status --short
git ls-files | rg '(^|/)(\.env|tmp|storage|backups)/' && exit 1 || true
git grep -nE 'admin123|haisi_dev_only|BEGIN (RSA|OPENSSH) PRIVATE KEY' -- . ':!docs/superpowers/plans/*' && exit 1 || true
```

Expected: no tracked secret, runtime, backup, or real report path.

- [ ] **Step 6: Commit the verified release candidate**

```bash
git add e2e README.md docs/operations/acceptance-report.md
git commit -m "test: verify customer archive release candidate"
```

---

## Final Verification Gate

Before calling the first version complete:

1. Run the complete Task 10 command set from a clean checkout.
2. Verify the working tree is clean.
3. Verify `git diff origin/main...HEAD --check` is clean.
4. Review the NocoBase license stored with the pinned distribution and record its exact commit/package provenance.
5. Push the `main` branch only after all checks pass and no real customer material is tracked.
