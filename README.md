# AuthN / AuthZ add-in (`rulebook-auth`)

An Effortless **add-in seed** (kind `child`), mounted at `auth/`. Requires the
`rulebook-backend` add-in. It has no rulebook of its own: it reads the
project's `../../effortless-rulebook/effortless-rulebook.json`.

## Requirement: the rulebook's Security module

**The build fails on a rulebook without the Security module.** `rulebook-to-rbac`
needs these tables in the rulebook, and stops with a message naming each one
that is missing:

- `ERBTables`, `ERBFields` (the Schema module, which Security requires), with
  one row per table and field of the rulebook;
- `ERBRoles` with **at least one role**, `ERBUsers`,
  `ERBRoleTablePermissions`, `ERBRoleFieldPermissions`, `ERBContextVariables`;
- `_meta.erb.modules.schema.enabled` and `_meta.erb.modules.security.enabled`
  set to `true`.

To add it: open the rulebook editor -> **Modules -> Security -> Enable** (it
creates the tables and sets both flags), press **Sync** to fill
`ERBTables`/`ERBFields`, add a role (e.g. `admin`, `DefaultCRUD` `CRUD`,
`IsAdminEquivalent` true), save, and build again. The table definitions are
`docs/ERB-MODEL.md` section 6 in Versioned-Stable-SSoTme-Tools.

## What the build produces

| Path | From | What it is |
|---|---|---|
| `rbac/effortless-rbac.json`, `rbac/rbac-report.md` | rulebook-to-rbac | The roles, permissions and row filters as a substrate-neutral model |
| `policies/06-rbac-policies.sql` | effortless-rbac-to-postgres-policies | Postgres roles, grants, `app.*` identity helpers and row-level security. Apply it after `backend/postgres` (it is numbered to follow `05-insert-data.sql`). |
| `policies/rbac-verify.sql` | effortless-rbac-to-postgres-policies | A hand-run check that no non-admin role sees a row without an identity |
| `magic-link.json` | this seed (filled from your answers) | Your magic-link tenant's send-code / verify-code endpoints, JWT issuer, and the tables that hold users |

## Sign-in

Authentication is passwordless email codes from `magiclink.effortlessapi.com`:
the app posts the email to `sendCodeUrl`, then the code to `verifyCodeUrl`,
and gets an RS256 JWT. Verify it with the tenant's public key
(`GET tenantInfoUrl` returns `public_key_pem`), pin `iss` to `jwtIssuer`, then
`SET LOCAL app.jwt = '<claims json>'` per transaction so the policies in
`06-rbac-policies.sql` see the identity. No secret is needed: the tenant id and
its public key are public.

## Questions

| Key | Default | Used for |
|---|---|---|
| `magicLinkTenantId` | (required) | `magic-link.json` endpoints and issuer |
| `userTables` | blank | `magic-link.json` `userTables`: the tables whose rows are users |

## How it is used

```bash
effortless cloneSeed effortlessapi/erb-seed-rulebook-auth auth
rm -rf auth/.git
cd auth && effortless build
```
