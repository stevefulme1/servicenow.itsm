# ServiceNow permissions required by this collection

To use the modules and plugins in `servicenow.itsm`, the account that
authenticates to your ServiceNow instance must be granted enough access on the
ServiceNow side. This page documents the roles that are typically required.

> **Important:** In ServiceNow, access is ultimately governed by
> [Access Control Lists (ACLs)](https://www.servicenow.com/docs/bundle/zurich-platform-security/page/administer/contextual-security/concept/access-control-rules.html),
> which every instance can customize. The roles listed here are the
> **out-of-the-box (OOB) defaults** and should be treated as guidance, not a
> guarantee. If a call authenticates successfully but returns empty results or a
> `403`/`User Not Authorized` error, the cause is almost always an ACL that the
> account does not satisfy. Use ServiceNow's *Debug Security* feature to see
> exactly which ACL check fails.

## The two layers of access

A request from this collection must pass **two independent checks**:

1. **REST API access.** The `snc_platform_rest_api_access` role controls whether
   an account may call the platform REST APIs (Table, Attachment, Import Set,
   Aggregate). The corresponding REST API ACLs are **inactive by default**, so on
   a stock instance no extra role is needed. If your administrator has activated
   the Table API ACL, the account additionally needs the
   `snc_platform_rest_api_access` role (it replaced the older `rest_service`
   role in the Kingston release).

2. **Table / record access.** Even with REST access granted, the account still
   needs a role (or a matching custom ACL) that permits read and/or write on the
   specific table a module operates on. This is the layer described in the table
   below.

## Role requirements per module

| Module(s) | ServiceNow table / API | Read | Create / Update / Delete |
| --------- | ---------------------- | ---- | ------------------------ |
| `incident`, `incident_info` | `incident` | `itil` | `itil` |
| `problem`, `problem_info` | `problem` | `itil` | `itil` |
| `problem_task`, `problem_task_info` | `problem_task` | `itil` | `itil` |
| `change_request`, `change_request_info` | `change_request` | `itil` | `itil` (`change_manager` for change management) |
| `change_request_task`, `change_request_task_info` | `change_task` | `itil` | `itil` |
| `catalog_request`, `catalog_request_info` | `sc_request` | `sn_request_read`, `sn_request_write`, `catalog`, or `itil` | `sn_request_write` or `itil` |
| `catalog_request_task`, `catalog_request_task_info` | `sc_task` | `sn_request_read`, `catalog`, or `itil` | `itil` (or a custom ACL) |
| `configuration_item`, `configuration_item_batch` | `cmdb_ci` and subclasses | `itil` | `itil` **or** `asset` |
| `configuration_item_info` | `cmdb_ci` and subclasses | `itil` | n/a (read only) |
| `configuration_item_relations` | `cmdb_rel_ci` | `itil` | `itil` **or** `asset` |
| `configuration_item_relations_info` | `cmdb_rel_ci` | `itil` | n/a (read only) |
| `service_catalog`, `service_catalog_info` | Service Catalog API (`/api/sn_sc/servicecatalog`) | access to the catalog / item | access to order the catalog item |
| `attachment_upload` | `sys_attachment` | — | write access on the **parent** record's table |
| `attachment_info` | `sys_attachment` | read access on the **parent** record's table | — |
| `api`, `api_info` | any table you target | depends on the target table | depends on the target table |

### Notes on specific modules

- **`itil` covers most ITSM tables.** Granting the `itil` role is the simplest
  way to give an integration account read/write access to incidents, problems,
  changes and their tasks. It is also the parent of the `sn_request_*` roles, so
  `itil` users can access catalog requests as well.

- **Catalog modules** (`catalog_request*`) can use the lower-privilege
  `sn_request_read` / `sn_request_write` roles instead of `itil` when you want to
  limit access to request tables only. Note that `sc_task` inherits field-level
  ACLs from the parent `task` table, so writing to catalog tasks with a custom
  role may require adding that role to those parent ACLs.

- **Configuration items / CMDB** require `itil` or `asset` to create, update or
  delete records in `cmdb_ci` and its subclasses. Be aware that, starting with
  the Xanadu release (Patch 9), the `sn_cmdb_editor` and `sn_cmdb_admin` roles
  **no longer** have create/update/delete access to `cmdb_ci`.

- **Attachments** inherit their permissions from the parent record. To upload an
  attachment to an incident, for example, the account needs write access to the
  `incident` table.

- **Service Catalog** (`service_catalog*`) uses a scripted REST API that mirrors
  the catalog UI, so access is governed by whether the account can see and order
  the relevant catalog items rather than by a dedicated table role.

- **`api` / `api_info`** are generic and can target any table, so the required
  role is whatever that table's ACLs demand.

## Recommended setup for an integration account

1. Create a dedicated ServiceNow user for automation and, on the user record,
   select **Web service access only** so the account cannot log in to the UI.
2. Grant the roles needed for the tables you automate — `itil` for broad ITSM
   coverage, or a **custom role with targeted read/write ACLs** for least
   privilege.
3. Add `snc_platform_rest_api_access` **if** the platform REST API ACL is active
   on your instance.
4. If you loosen ACLs to give a non-`itil` account access, review the licensing
   impact — widening record visibility can turn an account into a licensed user.

## References

- [Base system roles (Zurich)](https://www.servicenow.com/docs/bundle/zurich-platform-administration/page/administer/roles/reference/r_BaseSystemRoles.html)
- [Service Catalog roles](https://www.servicenow.com/docs/r/roles-by-product/roles_servicecatalog.html)
- [Service Catalog API reference](https://www.servicenow.com/docs/r/api-reference/rest-apis/c_ServiceCatalogAPI.html)
- [Roles required to access the Service Catalog (KB0778265)](https://support.servicenow.com/kb?id=kb_article_view&sysparm_article=KB0778265)
