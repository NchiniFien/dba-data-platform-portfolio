# SQL Server Always On: AG Replica Stuck in RESOLVING State

**Category:** SQL Server High Availability and Disaster Recovery  
**Technology:** SQL Server Always On Availability Groups, Windows Server Failover Clustering (WSFC)  
**Incident type:** Availability Group replica unable to transition to a healthy role  
**Portfolio classification:** Illustrative troubleshooting case study based on a supplied technical infographic  
**Last updated:** October 2026

> **Scope and confidentiality:** This is an educational case study, not a verified record of a specific production incident. Example server names, identities and log entries are illustrative. Never publish real employer infrastructure, account names, diagnostic logs or configuration details without authorization.

## SQL Server Always On: When RESOLVING Isn’t a Quorum Problem

> 🚨 **A healthy Windows Server Failover Cluster does not guarantee that every Always On Availability Group replica is healthy.**

What happens when your Always On Availability Group (AG) replica gets stuck in **RESOLVING**, even though the underlying cluster appears healthy?

This case-study scenario examines a replica that could not establish its expected AG role while the visible infrastructure checks appeared normal:

| Component checked | Observation |
|---|---|
| WSFC nodes | ✅ Both nodes online |
| Cluster quorum | ✅ Maintained |
| File Share Witness | ✅ Accessible |
| HADR endpoints | ✅ STARTED |
| Replica role/connection | ❌ One replica RESOLVING / DISCONNECTED |

The investigation therefore moved beyond a simple quorum or endpoint-status check and into the **Windows Server Failover Cluster (WSFC) diagnostic logs**.

### The clue that changed the investigation

```text
AccessCheck failed — 0x00000005 (Access Denied)
```

This error is consistent with an **access-control or authentication problem**. In the illustrated scenario, investigation focused on the relevant SQL Server service identity and its interaction with cluster resources. However, the error alone is **not proof** that a particular SQL Server account or permission was responsible: the failing security principal, requested action and target resource must be established from surrounding evidence.

### The six-step troubleshooting approach

1. **Confirm WSFC health** — quorum, witness, cluster nodes, roles and relevant resource status.
2. **Inspect the AG replica state** — query `sys.dm_hadr_availability_replica_states` and database-level health DMVs.
3. **Verify endpoint reachability and authorization** — a `STARTED` endpoint and successful TCP connection do not establish successful endpoint authentication.
4. **Correlate logs** — review the SQL Server error log, `alwayson_health` events and WSFC cluster logs at the same timestamps.
5. **Verify the actual security failure** — identify the account, denied operation and resource before changing any rights.
6. **Make a targeted change and validate recovery** — use approved least-privilege remediation and confirm that the replica returns to its intended role and expected synchronization health.

### Key takeaway

**RESOLVING is a symptom, not a root-cause diagnosis.** A healthy cluster does not necessarily mean an AG replica can perform every operation it needs, and an HADR endpoint in the STARTED state is not proof of an operational AG.

Before rebuilding an AG, forcing quorum or making disruptive configuration changes, follow the evidence. Sometimes the most revealing diagnostic information sits **outside the SQL Server engine — in WSFC logs**.

> **Portfolio transparency:** The source infographic and narrative describe a production-style incident, but independent production logs and post-change evidence were not provided with this portfolio write-up. This is therefore presented as an **illustrative technical case study**, not a firsthand verified claim of having fixed this incident in production. If you investigated it yourself, you can add sanitized firsthand evidence and update that classification accordingly.

## 1. Executive summary

An Always On Availability Group replica remains in the **RESOLVING** role while the Windows Server Failover Cluster appears healthy and the HADR endpoint appears started. A cluster diagnostic log shows **Access Denied (`0x00000005`)** during an access check. This points toward investigation of an authorization boundary, **not automatically toward a proven missing SQL Server service-account permission**.

The aim is to move methodically from symptoms to evidence, test the most likely hypotheses and validate a controlled fix without disrupting a healthy primary replica.

**Key learning:** Investigate the precise failed identity and resource access before changing permissions. Avoid treating RESOLVING as a single-cause symptom.

## 2. Observed symptoms

- One replica is reported as **RESOLVING** rather than transitioning to its expected PRIMARY or SECONDARY role.
- The replica does not achieve a healthy online or synchronization state.
- Initial checks indicate WSFC components, endpoints and basic network connectivity are available.
- Cluster diagnostics show an access or authentication failure (example HRESULT `0x00000005` (Win32 error 5; HRESULT form `0x80070005`)).

**Example role snapshot (illustrative):**

| Replica | Expected role | Observed role | Synchronization/health |
|---|---|---|---|
| SQL01 | PRIMARY | PRIMARY | Healthy/synchronized as appropriate |
| SQL02 | SECONDARY | RESOLVING | Not healthy; investigate |

> A RESOLVING replica does not have an established primary/secondary role at the time observed. Synchronization descriptions are database- and replica-specific; query the correct DMVs rather than treating the entire AG as one database-level status.

## 3. Investigation workflow

### Step A — Confirm availability group and replica health

Run on an appropriate AG-enabled SQL Server instance. Results depend on the instance queried and visibility/permissions.

```sql
-- Replica roles and operational health
SELECT
    ag.name AS availability_group_name,
    ar.replica_server_name,
    ars.role_desc,
    ars.operational_state_desc,
    ars.connected_state_desc,
    ars.recovery_health_desc,
    ars.synchronization_health_desc,
    ars.last_connect_error_number,
    ars.last_connect_error_description,
    ars.last_connect_error_timestamp
FROM sys.availability_groups AS ag
JOIN sys.availability_replicas AS ar
    ON ag.group_id = ar.group_id
LEFT JOIN sys.dm_hadr_availability_replica_states AS ars
    ON ar.replica_id = ars.replica_id
ORDER BY ag.name, ar.replica_server_name;
```

**Interpretation:** Review role, connected state, error text and timestamps. Run diagnostics on relevant nodes; remote replica values and visibility may differ by where you query.

```sql
-- Database-level synchronization and health
SELECT
    ag.name AS availability_group_name,
    ar.replica_server_name,
    DB_NAME(drs.database_id) AS database_name,
    drs.is_local,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.database_state_desc,
    drs.is_suspended,
    drs.suspend_reason_desc
FROM sys.dm_hadr_database_replica_states AS drs
JOIN sys.availability_replicas AS ar
    ON drs.replica_id = ar.replica_id
JOIN sys.availability_groups AS ag
    ON ar.group_id = ag.group_id
ORDER BY ag.name, ar.replica_server_name, database_name;
```

### Step B — Verify HADR endpoints and connectivity

```sql
SELECT
    e.name AS endpoint_name,
    e.state_desc AS endpoint_state,
    dme.role_desc AS mirroring_role,
    dme.connection_auth_desc,
    dme.encryption_algorithm_desc,
    te.port
FROM sys.database_mirroring_endpoints AS dme
JOIN sys.endpoints AS e
    ON dme.endpoint_id = e.endpoint_id
JOIN sys.tcp_endpoints AS te
    ON dme.endpoint_id = te.endpoint_id;
```

Confirm the relevant TCP port (often 5022, but not universally) and actual endpoint authentication mode. On Windows, test connectivity from each replica to the other, for example:

```powershell
Test-NetConnection -ComputerName SQL02 -Port 5022
```

**Important:** A successful TCP test establishes reachability, not that SQL Server endpoint authentication and authorization are correct.

### Step C — Examine WSFC membership and quorum

```sql
SELECT member_name, member_type_desc, member_state_desc, number_of_quorum_votes
FROM sys.dm_hadr_cluster_members;

SELECT cluster_name, quorum_type_desc, quorum_state_desc
FROM sys.dm_hadr_cluster;
```

Cross-check with **Failover Cluster Manager**, `Get-ClusterNode` and `Get-ClusterGroup` on the applicable Windows hosts. Healthy quorum does not rule out AG-specific authorization failures.

### Step D — Inspect SQL Server and cluster logs

Review:

1. SQL Server ERRORLOG and Windows Application/System events around the incident time.
2. Always On health Extended Events (including the `alwayson_health` session, where available).
3. WSFC cluster diagnostics, generated from an elevated PowerShell session with approved access:

```powershell
Get-ClusterLog -UseLocalTime -TimeSpan 30 -Destination C:\Temp\ClusterDiagnostics
```

Look for access failures, failed resource operations, service-account identity, SPNs/Kerberos/NTLM messages, endpoint connection errors and timing correlations.

**Illustrative evidence from the supplied infographic:** `AccessCheck failed` and `Access Denied (0x00000005)`. These fragments support further investigation of permissions/authentication but do not, by themselves, identify the resource or exact permission that failed.

### Step E — Identify the failed identity and security boundary

Determine **which identity** attempted **which operation** against **which resource**. Distinguish:

- SQL Server Database Engine service accounts and endpoint CONNECT permissions.
- Windows cluster service, cluster objects and relevant AD or local security rights.
- Availability Group cluster resource dependencies and ownership.
- Domain authentication, SPNs and network access where implicated by evidence.

Check actual SQL Server service identities using SQL Server Configuration Manager or approved inventory tooling. Validate endpoint permissions without blindly changing them:

```sql
SELECT
    ep.name AS endpoint_name,
    perm.state_desc,
    perm.permission_name,
    pr.name AS principal_name,
    pr.type_desc
FROM sys.server_permissions AS perm
JOIN sys.endpoints AS ep
    ON perm.major_id = ep.endpoint_id
JOIN sys.server_principals AS pr
    ON perm.grantee_principal_id = pr.principal_id
WHERE perm.class_desc = 'ENDPOINT'
ORDER BY ep.name, pr.name;
```

## 4. Root-cause hypothesis and verification

The infographic attributes the issue to a **security/authorization failure**: an identity required for cluster/AG operations reportedly lacked access. Treat this as a **hypothesis until confirmed against actual logs and effective permissions**.

Before remediation, document:

- Exact event timestamp and originating node.
- Failed identity/account or principal.
- The resource and operation denied.
- Permission or authentication requirement, supported by vendor documentation or security review.
- Why the denial prevents the replica from joining or assuming its role.

## 5. Controlled resolution procedure

1. Record the current AG state, replica roles, endpoint configuration and cluster health.
2. Obtain change approval and confirm a viable backup/recovery path.
3. Identify and correct **only the verified missing permission or authentication configuration**, following least privilege. Do **not** broadly grant local administrator, sysadmin or cluster rights as a generic fix.
4. Re-check the previously failing access operation and related log entries.
5. Where necessary, use a documented, supported reconnect/restart procedure after considering availability and failover impacts. Do not force quorum, force failover, remove replicas or restart production services solely because the role shows RESOLVING.
6. Verify stable replica role, endpoint connectivity, database-level synchronization health and application connectivity.
7. Record the final root cause and prevent recurrence through a configuration baseline and appropriate alerts.

## 6. Post-fix validation checklist

- [ ] Replica transitions out of RESOLVING to its intended role.
- [ ] SQL Server reports healthy AG replica connection and recovery state.
- [ ] Database-level synchronization health is appropriate for each database and commit mode.
- [ ] SQL Server and cluster logs no longer show the same access failure.
- [ ] Listener/application traffic and critical jobs work as expected.
- [ ] Backups are current; restore testing and recovery procedures remain valid.
- [ ] Any remediation changes are recorded, reviewed and monitored.

## 7. Alternative causes to rule out

RESOLVING may also appear with cluster membership/quorum instability, AG resource or lease failures, SQL Server startup/recovery, endpoint connectivity/authentication issues, database recovery problems or network disruptions. Do not infer an access-control root cause without matching evidence.

## 8. Lessons learned

1. **Cluster health is not AG health.** Validate role state, replica and database health separately.
2. **Running endpoint is not proof of successful authentication.** Port reachability and SQL endpoint authorization are separate tests.
3. **Read cluster and SQL logs together.** A correlated security failure is often more useful than a generic dashboard status.
4. **Fix the specific failure, not the symptom.** Least-privilege changes and verification are safer than broad grants or forceful failover.
5. **Document the recovery.** Record evidence, approved change, result and preventive controls.

## 9. Recommended official references

- [Always On availability groups documentation](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server)
- [Monitor availability groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/monitor-availability-groups-sql-server)
- [Windows Server Failover Clustering](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview)
- [TechBrothersIT — SQL Server DBA tutorial index](https://www.techbrothersit.com/2014/12/sql-server-dba-tutorial.html)

## 10. Portfolio note

For a public portfolio, attach only diagrams and screenshots that you own or have permission to reproduce. If using the supplied infographic, confirm ownership/reuse permission and remove any sensitive identifiers first. Mark all simulated test results clearly as **lab results**, not production outcomes.

---

**Suggested GitHub location:** `03-High-Availability-DR/AG-Replica-Stuck-In-Resolving-State/README.md`
