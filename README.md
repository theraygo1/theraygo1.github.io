# SQL Server DBA — 100 FAQ Reference

**Scope:**Installation & administration, environment management, code review/standards, data protection, performance tuning, SSIS, security, 

backup/NAS retention, production support,Argent job scheduling, data modeling, query assistance, execution plans/indexing, documentation, 

access control, backup/recovery, storage/archiving, capacity planning, 24x7 on-call, SDLC, and toolset (SSMS, Profiler, SSIS, SSRS, SSAS, BCP, Commvault, Perfmon, ARGENT, ERwin, VSS/TFS, Visio).

---

## Section 1 — Installation, Configuration & Maintenance

**Q1. What are the key steps to install SQL Server in an enterprise environment?**
- Validate hardware/OS prerequisites (CPU, RAM, storage tiering, Windows edition compatibility)
- Choose the correct SQL Server edition (Standard/Enterprise) based on licensing and workload
- Run installation via Configuration Manager or a scripted/silent install for repeatability
- Separate data, log, and tempdb files across dedicated volumes during setup
- Apply the latest Cumulative Update (CU) immediately after install
- Configure service accounts with least-privilege domain accounts (gMSA preferred)
- Document the build in the standard server build checklist

**Q2. How do you harden a SQL Server instance after installation?**
- Disable unnecessary protocols (leave only TCP/IP enabled as required)
- Enable encrypted connections (TLS 1.2+) for client-server traffic
- Rename or disable the sa account; enforce mixed-mode only when required
- Apply CIS/DISA STIG hardening baseline checklist
- Restrict xp_cmdshell and other surface-area features unless justified
- Configure Windows Firewall rules for the SQL Server port
- Enable auditing (Server Audit Specification) for security-relevant events

**Q3. What is the recommended approach for post-install configuration of memory settings?**
- Set max server memory leaving headroom for the OS and other services
- Leave min server memory at default unless co-hosting requires floor protection
- Configure max degree of parallelism (MAXDOP) per Microsoft's NUMA-based guidance
- Set cost threshold for parallelism above the 5 default (commonly 25–50)
- Validate Lock Pages in Memory privilege is granted to the service account
- Monitor Page Life Expectancy post-configuration to validate memory pressure
- Re-baseline after major workload changes

**Q4. How do you configure tempdb correctly on a new instance?**
- Create multiple tempdb data files (commonly matching logical CPU count up to 8)
- Size all tempdb files equally with equal autogrowth settings
- Place tempdb on the fastest available storage, separate from user databases
- Enable trace flag 1117/1118 behavior (built-in by default from SQL 2016+)
- Pre-size tempdb files to avoid frequent autogrowth events
- Monitor tempdb contention via sys.dm_os_waiting_tasks (PAGELATCH waits)
- Revisit sizing after significant workload or tempdb-heavy feature additions

**Q5. What ongoing maintenance tasks keep an instance healthy?**
- Schedule regular index maintenance (rebuild/reorganize based on fragmentation)
- Update statistics on a recurring schedule and after large data loads
- Run DBCC CHECKDB regularly to detect corruption early
- Monitor and manage transaction log growth and VLF counts
- Patch with CUs/security updates on a defined maintenance cadence
- Review error logs and SQL Agent job history routinely
- Validate backup jobs are completing successfully every cycle

---

## Section 2 — Environment Management (Dev/Staging/Training/Production)

**Q6. How do you maintain consistency across Dev, Staging, Training, and Production environments?**
- Use source-controlled deployment scripts (TFS/VSS) --> (Team Foundation Server and Visual SourceSafe) for schema changes
- Apply the same patch level and configuration baseline across all tiers
- Use environment-specific config files rather than hardcoded connection strings
- Refresh lower environments periodically from sanitized production backups
- Document environment-specific deltas (server names, linked servers, jobs)
- Automate environment comparison using schema-compare tooling
- Gate promotion between environments through a change-approval process

**Q7. What is the correct promotion path for a database change?**
- Develop and unit test the change in Development
- Peer review the code/change against coding standards
- Deploy to Staging for integration and regression testing
- Validate in Training with representative data volumes
- Obtain change approval/CAB sign-off before Production deployment
- Deploy to Production during an approved maintenance window
- Confirm post-deployment validation and rollback readiness

**Q8. How do you sanitize production data before refreshing lower environments?**
- Identify PII/PHI (PII (Personally Identifiable Information) and PHI (Protected Health Information)) columns subject to data protection requirements
- Apply masking or tokenization scripts during the refresh process
- Exclude or truncate audit/log tables not needed for testing
- Restrict access to the refreshed environment based on need-to-know
- Validate that masking scripts run successfully as part of the refresh job
- Document what is masked for compliance/audit purposes
- Re-apply environment-specific permissions after refresh

**Q9. How should differing performance characteristics between environments be handled?**
- Right-size lower environments to approximate production I/O patterns where feasible
- Document known performance deltas so testing results are interpreted correctly
- Use representative data volumes rather than tiny subsets for realistic testing
- Avoid testing performance-sensitive changes only on undersized Dev boxes
- Use Staging as the closest proxy to Production for load/performance testing
- Track configuration drift between environments as a monitored item
- Re-test on Production-equivalent hardware before high-risk releases

**Q10. What controls prevent unauthorized changes from reaching Production?**
- Restrict direct Production write access to a small DBA group
- Require change tickets/approvals for all schema and config changes
- Use deployment pipelines rather than manual ad hoc scripts in Production
- Log and audit all changes made outside the standard pipeline
- Separate duties between developers (code) and DBAs (deployment)
- Review deployment scripts against the SDLC change record before execution
- Maintain a rollback script for every Production deployment

---

## Section 3 — Code Review, Optimization & Coding Standards

**Q11. What do you check for during a T-SQL code review?**
- Adherence to ANSI SQL standards and organizational naming conventions
- Absence of SELECT * in production code
- Proper use of set-based logic instead of row-by-row (RBAR) processing
- Correct transaction handling and error handling (TRY/CATCH)
- Appropriate indexing support for the query patterns introduced
- SARGability of WHERE clause predicates
- Avoidance of implicit conversions and non-deterministic functions in joins

**Q12. How do you optimize a poorly performing stored procedure?**
- Capture the actual execution plan and identify the costliest operators
- Check for missing or unused indexes supporting the query
- Look for parameter sniffing symptoms and evaluate OPTION(RECOMPILE) or local variables
- Review for scalar UDFs that could be inlined or rewritten as set-based logic
- Break down large multi-statement procedures into simpler, testable units
- Validate statistics are current on tables involved
- Re-test with representative data volumes, not just Dev-scale data

**Q13. What ANSI SQL compliance issues are commonly flagged in review?**
- Use of non-ANSI join syntax (old-style comma joins with WHERE-based joins)
- Implicit data type conversions that bypass SARGability
- Non-standard proprietary functions where ANSI equivalents exist
- Missing explicit column lists in INSERT statements
- Inconsistent NULL handling (ISNULL vs COALESCE usage standards)
- Lack of explicit transaction boundaries for multi-statement operations
- Inconsistent quoting/identifier standards (QUOTED_IDENTIFIER settings)

**Q14. How do you enforce coding standards across a DBA/dev team?**
- Publish and maintain a living coding standards SOP document
- Use code review checklists integrated into the pull request process
- Provide linting or static analysis tooling where available
- Run periodic training sessions for junior developers/DBAs
- Track recurring standard violations to target coaching
- Include standards compliance as part of release readiness criteria
- Update the standards document as new SQL Server features are adopted

**Q15. What's the process for identifying and fixing anti-patterns in legacy code?**
- Inventory legacy procedures via query store or plan cache analysis
- Prioritize by resource consumption (CPU/logical reads) and execution frequency
- Rewrite cursors and iterative logic into set-based equivalents where feasible
- Modernize deprecated syntax and features flagged by Database Engine Tuning Advisor
- Validate business logic parity through regression testing before replacing legacy code
- Document the rationale and before/after performance metrics
- Schedule remediation incrementally to limit regression risk

---

## Section 4 — Data Standards & Data Protection Compliance

**Q16. How do you maintain data standards across the enterprise?**
- Maintain a data dictionary defining naming, types, and permissible values
- Enforce consistent naming conventions for tables, columns, and constraints
- Standardize data types for common domains (dates, currency, identifiers)
- Require documentation for any schema deviation from the standard
- Periodically audit databases against the published data standards
- Involve data governance stakeholders in standards updates
- Version-control the data standards document itself

**Q17. What steps ensure compliance with data protection regulations?**
- Classify data by sensitivity (PII, PHI, financial, public) -- PII (Personally Identifiable Information) and PHI (Protected Health Information)
- Apply encryption at rest (TDE) and in transit (TLS-Transport Layer Security) for sensitive data
- Implement column-level encryption or Dynamic Data Masking where appropriate
- Restrict access to sensitive data via role-based permissions
- Maintain audit logs of access to regulated data
- Define and enforce data retention/purge schedules per regulatory requirement
- Review compliance posture during periodic internal/external audits

**Q18. How do you handle a request to purge data under a retention policy?**
- Confirm the applicable retention period from policy/legal documentation
- Identify all locations the data resides (primary tables, archives, backups)
- Execute purges through auditable, reviewed scripts rather than ad hoc deletes
- Coordinate with application owners to confirm downstream dependency impact
- Preserve backups per retention rules even after live-data purge
- Log the purge action with timestamp, scope, and approver
- Validate purge completeness with a post-purge verification query

**Q19. What is Dynamic Data Masking and when should it be used?**
- A feature that obfuscates sensitive column data for non-privileged users at query time
- Useful for masking data in lower environments or for support-tier access
- Does not encrypt data at rest — it only affects presentation to unauthorized roles
- Should be combined with proper permissions, not used as the sole control
- Configured per-column with masking functions (default, email, partial, random)
- Privileged roles (UNMASK permission) can view full unmasked data
- Should be documented as part of the data protection control set

**Q20. How do you audit and report on data access for compliance purposes?**
- Enable SQL Server Audit or Extended Events to capture access events
- Define audit specifications scoped to sensitive objects only, to limit overhead
- Route audit logs to a centralized, tamper-evident location
- Build scheduled reports summarizing access patterns for compliance review
- Alert on anomalous access patterns (off-hours, bulk exports)
- Retain audit logs per the organization's retention policy
- Periodically test that auditing is actually capturing expected events

---

## Section 5 — Performance Monitoring & Query/Index Tuning

**Q21. How do you use SQL Profiler for performance troubleshooting?**
- Create a targeted trace capturing only relevant events to limit overhead
- Filter by database, duration, or application to reduce trace volume
- Capture RPC:Completed and SQL:BatchCompleted events for query duration/reads
- Correlate trace output with wait statistics for root-cause analysis
- Avoid running Profiler traces long-term on Production; prefer Extended Events
- Export trace results for offline analysis in Database Engine Tuning Advisor
- Stop and remove traces promptly after data collection completes

**Q22. How does the Database Engine Tuning Advisor (DBETA) help with tuning?**
- Analyzes a captured workload (trace or query) to recommend indexes
- Suggests index consolidations and removal of redundant indexes
- Recommends partitioning strategies for large tables
- Provides an estimated performance improvement percentage
- Should be validated against a representative workload, not a single query
- Recommendations should be reviewed manually before applying to Production
- Works best combined with execution plan analysis for context

**Q23. What are the common causes of SQL Server performance issues?**
- Missing or poorly designed indexes causing table/index scans
- Outdated statistics leading to poor cardinality estimates
- Parameter sniffing causing inconsistent plan performance
- Blocking and lock contention from long-running transactions
- Insufficient memory causing buffer pool pressure and PLE drops
- I/O subsystem bottlenecks (high disk latency)
- Poorly written queries (RBAR logic (Row-By-Agonizing-Row), unnecessary cursors, non-SARGable predicates)

**Q24. How do you identify and resolve blocking issues?**
- Query sys.dm_exec_requests / sys.dm_os_waiting_tasks to find blocking chains
- Identify the head blocker and its executing statement
- Review the isolation level in use and consider READ COMMITTED SNAPSHOT
- Check for missing indexes causing lock escalation on large scans
- Shorten long-running transactions and batch large DML operations
- Set appropriate lock timeout / deadlock priority where applicable
- Monitor recurring blocking patterns via Extended Events for trend analysis

**Q25. What's your approach to proactive performance monitoring?**
- Establish baseline metrics for CPU, memory, I/O, and wait statistics
- Use Query Store to track query performance regressions over time
- Set up alerts for key thresholds (PLE, CPU, blocking duration)
- Review top resource-consuming queries on a recurring cadence
- Track index fragmentation and usage statistics regularly
- Use Perfmon counters alongside SQL-level DMVs for OS-level correlation
- Document findings and remediation actions in a performance log

---

## Section 6 — SSIS Development, Scheduling & Deployment

**Q26. What are best practices for developing SSIS packages?**
- Use parameterized connection managers instead of hardcoded values
- Implement logging (SSIS catalog logging or custom logging tables)
- Use checkpoints or restartability logic for long-running packages
- Handle errors with dedicated error output paths, not silent failures
- Modularize logic into reusable packages/tasks where possible
- Optimize data flow buffer sizes for large volume loads
- Document package purpose, inputs, and dependencies

**Q27. How do you deploy SSIS packages to a production environment?**
- Build and validate the package in the project deployment model
- Deploy to the SSIS Catalog (SSISDB) rather than legacy file-system deployment
- Configure environment-specific parameters via SSISDB environments
- Validate package execution with a test run before scheduling
- Set appropriate logging level for production execution
- Version the deployment and retain rollback capability
- Grant execute permissions only to required service accounts

**Q28. How are SSIS packages scheduled as SQL Agent jobs?**
- Create a SQL Agent job with an "Execute SSIS Package" job step
- Reference the SSISDB-deployed package and its environment reference
- Configure job schedules aligned with upstream/downstream data dependencies
- Set up job step failure notifications and retry logic
- Use job categories to organize ETL jobs logically
- Log job history and package execution reports for troubleshooting
- Test the scheduled execution end-to-end before go-live

**Q29. How do you troubleshoot a failing SSIS package in production?**
- Review the SSISDB execution report for the failed step and error message
- Check data flow-level error outputs for row-level failure detail
- Validate source/destination connectivity and credentials
- Confirm upstream data availability and format hasn't changed
- Re-run the package in Verbose logging mode to capture more detail
- Check for environment parameter mismatches between Dev and Prod
- Document the root cause and apply a permanent fix, not just a rerun

**Q30. What performance considerations apply to large-volume SSIS data loads?**
- Use fast-load/bulk insert options on OLE DB/ADO.NET destinations
- Disable unnecessary indexes/constraints during bulk load, rebuild after
- Tune the DefaultBufferMaxRows/DefaultBufferSize for the data flow
- Avoid unnecessary sorts and blocking transformations in the data flow
- Partition large loads into batches to control transaction log growth
- Run loads during low-activity windows for minimal contention
- Monitor tempdb and log growth during large ETL runs

---

## Section 7 — Security: Logins, Users, Roles & Authentication

**Q31. How do you implement least-privilege access via SQL logins and roles?**
- Create custom database roles mapped to job function, not individual grants
- Avoid granting sysadmin/db_owner except where explicitly justified
- Use Windows Authentication/AD groups over SQL logins where possible
- Grant object-level permissions through role membership, not direct grants
- Periodically review role membership against current job responsibilities
- Document the rationale for any elevated access granted
- Remove access promptly upon role change or termination

**Q32. What's the difference between Windows Authentication and Mixed Mode, and when is each used?**
- Windows Authentication relies on AD/Kerberos, offering centralized credential management
- Mixed Mode also allows SQL logins with server-managed passwords
- Windows Authentication is preferred for internal enterprise applications
- Mixed Mode may be required for legacy apps or non-domain-joined clients
- SQL logins require strong password policy enforcement (CHECK_POLICY)
- Mixed Mode increases the attack surface and should be limited where possible
- Authentication mode changes require a service restart to take effect

**Q33. How do you audit and manage login/role changes over time?**
- Use SQL Server Audit or DDL triggers to capture login/role change events
- Maintain a change log correlating access grants to approval tickets
- Run periodic access recertification reviews with data/application owners
- Flag and investigate any out-of-band permission changes
- Compare current role membership against a baseline periodically
- Revoke unused or orphaned logins found during review
- Report audit findings to security/compliance stakeholders

**Q34. How do you secure service accounts used by SQL Server services?**
- Use Group Managed Service Accounts (gMSA) where supported
- Apply least-privilege OS and domain permissions to the service account
- Rotate credentials per organizational password policy (or rely on gMSA auto-rotation)
- Avoid using the same service account across multiple unrelated services
- Restrict interactive logon rights for service accounts
- Monitor service account usage for anomalous activity
- Document service account ownership and purpose

**Q35. What steps do you take to resolve orphaned users after a database restore/migration?**
- Identify orphaned users via sp_change_users_login or sys.sp_validatelogins query pattern
- Map orphaned users to existing or newly created server logins
- Use ALTER USER...WITH LOGIN to relink the user to the correct login
- Recreate missing logins with matching SIDs when migrating between instances
- Validate role membership survived the relinking process
- Test application connectivity after remediation
- Document the fix as part of the migration runbook for future migrations

---

## Section 8 — NAS Backups & Retention Policies

**Q36. How do you configure and monitor NAS-based backups?**
- Direct SQL Server backup jobs to a NAS share with adequate throughput/IOPS
- Validate network path availability and permissions before scheduling
- Monitor backup job duration and success/failure via SQL Agent alerts
- Periodically test restore from NAS-stored backups to validate integrity
- Monitor NAS capacity thresholds to prevent backup failures from disk-full conditions
- Use compression to reduce backup size and network transfer time
- Maintain a backup job dashboard/report reviewed daily

**Q37. How do you align backup retention with state and federal requirements?**
- Determine the applicable retention period from regulatory/legal guidance
- Configure backup retention/cleanup jobs to match the mandated period exactly
- Retain backups longer than the operational recovery need when regulation requires it
- Separate short-term operational backups from long-term compliance archives
- Document the retention schedule in the DBA SOP library
- Periodically audit that retention jobs are enforcing the policy correctly
- Coordinate with legal/compliance on any retention policy changes

**Q38. What backup strategy do you use for critical production databases?**
- Full backups on a defined schedule (e.g., nightly) as the baseline
- Differential backups between fulls to reduce recovery time and log growth
- Transaction log backups at short intervals to minimize RPO
- Store backups on NAS with a secondary offsite/cloud copy for DR
- Validate backup chain integrity regularly (RESTORE VERIFYONLY / test restores)
- Encrypt backups containing sensitive/regulated data
- Document RPO/RTO targets achieved by the strategy per database tier

**Q39. How do you handle backup failures on the NAS target?**
- Review the SQL Agent job history and error log for the failure reason
- Check NAS availability, capacity, and network connectivity first
- Re-run the backup manually once the root cause is resolved
- Escalate to infrastructure/storage team if the issue is NAS-side
- Ensure the transaction log doesn't grow unbounded during the outage
- Notify stakeholders if RPO is at risk due to the failure
- Document the incident and any process improvement identified

**Q40. How do you validate that backup and retention procedures remain effective?**
- Perform periodic test restores to a non-production environment
- Verify backup file checksums/integrity as part of the validation job
- Confirm retention cleanup jobs are removing only expired backups correctly
- Review backup completion and duration trends for early warning signs
- Reconcile backup inventory against the expected schedule regularly
- Update the DR runbook based on validation findings
- Report validation results to management/compliance as required

---

## Section 9 — Production Issue Analysis & Help Desk Ticket Resolution

**Q41. What is your process for triaging a production issue from a help desk ticket?**
- Gather ticket details: affected system, error message, time of occurrence, user impact
- Reproduce or correlate the issue with SQL Server logs/Agent job history
- Classify severity/priority based on business impact
- Identify whether the issue is application, network, or database-layer
- Apply an immediate mitigation if available while root cause is investigated
- Communicate status updates to the ticket owner/stakeholders
- Document root cause and resolution in the ticket and knowledge base

**Q42. How do you diagnose a "database is slow" ticket with no specifics?**
- Ask for specific symptoms: timeframe, affected query/screen, error messages
- Check current wait statistics and active sessions for anomalies
- Review recent changes (deployments, index changes, data volume growth)
- Correlate with Perfmon/DMV data for the reported time window
- Check for blocking, long-running queries, or resource contention
- Compare against established performance baselines
- Provide findings and remediation steps back to the requester

**Q43. What's your approach to resolving a recurring intermittent issue?**
- Set up proactive monitoring/alerting to catch the issue in the act
- Correlate occurrences with job schedules, batch windows, or peak load times
- Use Extended Events sessions to capture detail without full-time overhead
- Review historical Query Store data for regression patterns
- Test hypotheses in a controlled environment before changing Production
- Implement a fix and monitor for recurrence over an agreed period
- Close the loop with a documented root cause, not just a workaround

**Q44. How do you prioritize multiple simultaneous production issues?**
- Assess business impact and number of affected users/systems for each
- Apply the organization's defined severity/priority matrix
- Communicate expected response times to all ticket owners
- Delegate or escalate lower-priority items if capacity is constrained
- Address data-integrity or security-related issues with top priority
- Keep stakeholders updated as priorities shift
- Conduct a post-incident review once all issues are resolved

**Q45. What documentation do you produce after resolving a production incident?**
- Root cause analysis summary
- Timeline of detection, escalation, and resolution
- Technical fix details and any scripts/changes applied
- Business impact assessment (downtime, affected users)
- Preventive recommendations to avoid recurrence
- Update to the relevant SOP/knowledge base article
- Sign-off/closure notes in the ticketing system

---

## Section 10 — Batch Job Scheduling with Argent Job Scheduler / Queue Engine

**Q46. How do you design a reliable nightly batch job schedule?**
- Map job dependencies to ensure correct execution sequence
- Stagger jobs to avoid resource contention during peak batch windows
- Build in retry logic for transient failures
- Set realistic timeout thresholds per job based on historical duration
- Configure failure notifications routed to the on-call DBA
- Validate upstream data availability before dependent jobs trigger
- Maintain a batch schedule document listing all jobs and dependencies

**Q47. How does Argent Queue Engine help manage job dependencies?**
- Queues jobs in a defined sequence to prevent overlapping resource-heavy tasks
- Ensures dependent jobs wait for predecessor completion before starting
- Provides centralized visibility into queued/running/completed job status
- Helps throttle concurrent job execution to protect server resources
- Supports conditional execution based on prior job outcomes
- Reduces manual coordination needed for multi-step batch chains
- Integrates alerting when queue processing stalls or fails

**Q48. What steps do you take when a nightly batch job fails?**
- Review the Argent job log/error output for the failure point
- Determine whether it's transient (network blip) or a real data/logic issue
- Check if downstream dependent jobs should be held pending resolution
- Re-run the failed step manually once the cause is addressed
- Notify stakeholders if the failure impacts morning data availability (SLA risk)
- Document the failure and resolution for trend tracking
- Adjust retry/alerting thresholds if the failure pattern recurs

**Q49. How do you monitor batch job health across the enterprise?**
- Use Argent's dashboard/console for real-time job status visibility
- Configure alerts for jobs exceeding expected duration (SLA breach warning)
- Review a daily batch summary report each morning
- Track historical job duration trends to spot gradual degradation
- Correlate batch job failures with underlying SQL Server resource metrics
- Maintain an escalation path for critical batch job failures
- Periodically audit job schedules for stale or unnecessary jobs

**Q50. How do you plan capacity for growing batch processing windows?**
- Trend batch job duration over time against data volume growth
- Identify jobs approaching or exceeding their available window
- Evaluate parallelization or incremental-load redesign for long-running jobs
- Assess whether hardware/storage upgrades are needed to maintain SLA
- Negotiate adjusted batch windows with business stakeholders if needed
- Pilot optimizations in Staging before rolling into the nightly schedule
- Document capacity findings in the periodic capacity planning review

---

## Section 11 — Data Modeling

**Q51. What's your process for creating a physical data model?**
- Gather requirements from the logical data model and business rules
- Define tables, columns, data types, and constraints reflecting those rules
- Design primary/foreign key relationships enforcing referential integrity
- Apply appropriate normalization (typically 3NF) balanced against performance needs
- Select indexing strategy aligned with expected query patterns
- Document the model in ERwin (or equivalent) with clear naming standards
- Review the model with stakeholders before implementation

**Q52. How do you modify a physical data model without breaking existing applications?**
- Assess the impact of the change on dependent applications/reports
- Use additive changes (new columns/tables) where possible instead of destructive ones
- Provide backward-compatible views for consumers not yet updated
- Coordinate deployment timing with application release cycles
- Update the ERwin model and data dictionary to reflect the change
- Test the change thoroughly in Staging with representative data
- Communicate the schema change to all downstream stakeholders in advance

**Q53. When do you choose normalization versus denormalization in a design?**
- Normalize OLTP systems to reduce redundancy and maintain data integrity
- Denormalize reporting/analytics structures for query performance
- Consider read/write ratio: write-heavy systems favor normalization
- Use indexed views or materialized aggregates instead of full denormalization where possible
- Balance maintenance overhead against query performance gains
- Document the rationale for any deliberate denormalization decision
- Revisit the decision if workload patterns change significantly

**Q54. How does ERwin support ongoing data model governance?**
- Maintains a single source of truth for logical and physical models
- Supports forward/reverse engineering to keep models and databases in sync
- Provides naming standards and domain enforcement across models
- Enables model versioning and change comparison over time
- Generates documentation/data dictionary output automatically
- Supports subject area organization for large, complex schemas
- Facilitates collaborative review before schema changes are implemented

**Q55. What data modeling considerations apply to high-volume transactional tables?**
- Choose narrow, efficient data types to minimize row size
- Design indexes carefully to avoid excessive write overhead
- Consider partitioning strategies for very large tables
- Avoid unnecessary nullable columns and wide row overhead
- Plan for archiving/purging strategy from the outset
- Evaluate whether natural or surrogate keys best fit the access pattern
- Model with concurrency (locking/blocking impact) in mind, not just storage

---

## Section 12 — Assisting with Complex Queries

**Q56. How do you approach writing a complex multi-join analytical query?**
- Clarify the exact business requirement and expected output before writing SQL
- Break the query into logical CTEs for readability and testability
- Validate join cardinality at each step to avoid fan-out/duplication errors
- Use window functions instead of self-joins/subqueries where appropriate
- Test against a known-good sample dataset to validate correctness
- Review the execution plan for unexpected scans or spills
- Document the query's purpose and assumptions for future maintainers

**Q57. How do you help a developer debug a query returning incorrect results?**
- Reproduce the issue with the exact query and parameters provided
- Isolate each join/filter incrementally to locate where results diverge
- Check for NULL-handling issues in joins and aggregations
- Verify grouping/aggregation logic matches the intended business rule
- Confirm the data itself matches expectations (source data validation)
- Explain the root cause clearly, not just supply a fixed query
- Recommend a unit test or validation query to prevent recurrence

**Q58. What techniques help simplify deeply nested subqueries?**
- Convert nested subqueries into CTEs for clarity and step-by-step validation
- Replace correlated subqueries with JOINs or window functions where possible
- Use APPLY operators for row-by-row derived logic instead of subqueries
- Break the logic into temp tables for very complex multi-stage transformations
- Validate each simplified stage against the original query's output
- Comment complex logic clearly for future readability
- Re-check the execution plan after simplification for performance impact

**Q59. How do you assist with cross-database or cross-server complex queries?**
- Confirm linked server or cross-database permissions are correctly configured
- Evaluate performance implications of remote joins versus local staging
- Consider staging remote data locally for large-volume cross-server operations
- Validate data type/collation compatibility across sources
- Use OPENQUERY for pass-through execution when appropriate for performance
- Monitor for distributed query plan inefficiencies
- Document any cross-environment dependencies introduced

**Q60. What's your review checklist before approving a complex query for production use?**
- Correctness validated against expected business logic and sample results
- Execution plan reviewed for scans, spills, or missing index warnings
- Appropriate indexing exists to support the query pattern
- Query is parameterized/sargable, avoiding implicit conversions
- Resource impact assessed against concurrent production workload
- Error handling included if the query is part of a larger procedure
- Documentation/comments included for maintainability

---

## Section 13 — Execution Plans & Index Design

**Q61. How do you read an execution plan to find performance problems?**
- Identify the highest-cost operators by relative cost percentage
- Look for table/index scans on large tables where seeks are expected
- Check for large gaps between estimated and actual row counts
- Identify key lookups indicating a missing covering index
- Watch for spills to tempdb (sort/hash warnings) indicating memory pressure
- Review for implicit conversion warnings affecting SARGability
- Correlate plan findings with actual wait statistics for confirmation

**Q62. How do you decide which indexes to create based on execution plan analysis?**
- Identify columns used in WHERE, JOIN, and ORDER BY clauses causing scans
- Evaluate missing index suggestions from the plan, but validate independently
- Design covering indexes to eliminate key lookups for critical queries
- Consider column order in composite indexes based on selectivity and usage
- Avoid creating redundant or overlapping indexes
- Weigh index benefit against write-overhead cost for high-DML tables
- Test the new index's impact on the plan before deploying to Production

**Q63. What's the difference between a clustered and non-clustered index, and how does that affect design decisions?**
- A clustered index physically orders table data by its key
- A table can have only one clustered index; choose the key carefully
- Non-clustered indexes are separate structures pointing back to the clustered key (or RID)
- Clustered key choice affects the size/efficiency of every non-clustered index
- Narrow, static, ever-increasing keys generally make efficient clustered keys
- Non-clustered indexes support additional query patterns without reordering data
- Over-indexing with non-clustered indexes increases write and maintenance overhead

**Q64. How do you identify and remove unused or redundant indexes?**
- Query sys.dm_db_index_usage_stats to find low/no seek-usage indexes
- Identify indexes with high user_updates but low user_seeks/user_scans (pure overhead)
- Check for overlapping indexes with the same or near-identical key columns
- Validate usage stats over a full business cycle before removing (avoid a too-short sample)
- Disable rather than drop initially, to allow quick rollback if needed
- Monitor performance after removal to confirm no regression
- Document removed indexes and rationale for audit purposes

**Q65. How do you troubleshoot a query where the optimizer chooses a suboptimal plan?**
- Check statistics freshness on all tables/indexes involved
- Evaluate for parameter sniffing and test with different parameter values
- Review for implicit conversions preventing index usage
- Check MAXDOP/cost threshold settings affecting parallelism decisions
- Consider plan hints (OPTIMIZE FOR, RECOMPILE) as a targeted, documented fix
- Compare estimated vs. actual plans to pinpoint the cardinality mismatch
- Validate the fix doesn't negatively affect other query patterns sharing the plan

---

## Section 14 — Documentation & Metadata

**Q66. What core documents make up a complete DBA documentation set?**
- Data standards and naming convention guide
- Data dictionary / metadata repository
- Backup and recovery procedures (SOPs)
- Security and access control policy documentation
- Disaster recovery runbook
- Server build and configuration checklist
- Change management and deployment procedures

**Q67. How do you maintain an accurate data dictionary (metadata)?**
- Populate extended properties or a metadata repository for each table/column
- Keep definitions synchronized with the physical data model (ERwin)
- Establish a review cadence to catch drift between schema and documentation
- Require metadata updates as part of the schema change process
- Make the data dictionary accessible to developers and analysts
- Include business definitions, not just technical data types
- Version the data dictionary alongside schema version changes

**Q68. What should a well-written SOP for a recurring DBA task include?**
- Purpose and scope of the procedure
- Prerequisites and required permissions/tools
- Step-by-step instructions in clear, point-wise format
- Expected outcomes and validation steps
- Rollback/troubleshooting guidance for common failure points
- Ownership/author and last-reviewed date
- Related references (scripts, SOPs, escalation contacts)

**Q69. How do you keep documentation current as systems evolve?**
- Tie documentation updates to the change management process
- Schedule periodic documentation reviews independent of active changes
- Assign ownership for each document to a responsible team member
- Use version control for documentation, not just static file copies
- Solicit feedback from junior DBAs using the docs to find gaps
- Archive outdated documents clearly rather than leaving them ambiguous
- Track documentation debt as a visible backlog item

**Q70. Why is thorough documentation important beyond compliance requirements?**
- Reduces onboarding time for new or junior DBAs
- Preserves institutional knowledge when staff transition or are unavailable
- Speeds up incident response by providing known-good runbooks
- Supports consistent execution of recurring tasks across the team
- Provides an audit trail demonstrating due diligence for compliance
- Enables knowledge transfer during mentoring
- Reduces reliance on tribal knowledge held by a single individual

---

## Section 15 — Access Permissions & Privileges Control

**Q71. How do you design a role-based access control (RBAC) model for a database?**
- Define roles that map to business functions, not individuals
- Grant permissions at the role level; assign users to roles
- Apply least-privilege by default, escalating only with justification
- Separate read-only, read-write, and administrative roles clearly
- Periodically review role definitions against actual usage patterns
- Document each role's purpose and permission scope
- Avoid ad hoc, one-off permission grants outside the RBAC model

**Q72. How do you handle temporary elevated access requests?**
- Require a documented business justification and time-bound approval
- Grant the minimum permission necessary to complete the task
- Set an expiration/reminder to revoke access automatically or manually
- Log the grant and subsequent revocation for audit purposes
- Avoid granting sysadmin/db_owner for tasks achievable with narrower roles
- Notify the requester and their manager of the access window
- Review elevated access logs periodically for pattern anomalies

**Q73. What is your process for a periodic access recertification review?**
- Export current role/permission assignments across all databases
- Circulate the list to data/application owners for validation
- Flag and remove access no longer aligned with current job duties
- Document the review cycle completion for compliance evidence
- Investigate any access grants without a clear owner or justification
- Update the RBAC documentation to reflect any changes made
- Schedule the next recertification cycle

**Q74. How do you prevent privilege creep over time?**
- Tie access reviews to a recurring calendar cadence, not ad hoc only
- Require re-justification of access rather than automatic renewal
- Remove access promptly upon role change, not just termination
- Avoid granting broad permissions "just in case" future need arises
- Track and alert on manual out-of-process permission changes
- Educate requesters on requesting the narrowest role that meets the need
- Report privilege trends to management as part of governance reporting

**Q75. How do you audit who has access to a sensitive dataset?**
- Query role membership and explicit grants against the relevant objects
- Trace access through nested roles/group memberships, not just direct grants
- Cross-reference with SQL Server Audit logs for actual access history
- Validate that access aligns with documented business need
- Flag any access outside expected departments/roles for investigation
- Document findings for the compliance/security team
- Remediate excess access identified during the audit

---

## Section 16 — Backup & Recovery Plan Development and Testing

**Q76. How do you develop a backup and recovery plan for a new database?**
- Determine RPO/RTO requirements with the business/application owner
- Select the backup model (Full/Bulk-Logged/Simple) matching recovery needs
- Define full, differential, and log backup schedule intervals
- Choose backup storage location(s) including offsite/DR copies
- Document the exact restore sequence and expected recovery time
- Test the full restore process before go-live
- Review and update the plan whenever RPO/RTO requirements change

**Q77. How often should recovery plans be tested, and what does testing involve?**
- Test restores on a regular defined cadence (e.g., monthly/quarterly)
- Restore to an isolated, non-production environment to avoid impact
- Validate database consistency (DBCC CHECKDB) post-restore
- Time the restore process against the documented RTO target
- Test point-in-time recovery scenarios using log backups, not just full restores
- Document test results, issues found, and corrective actions
- Update the recovery runbook based on lessons learned from testing

**Q78. What's involved in a full disaster recovery (DR) exercise versus a routine restore test?**
- DR exercises simulate broader infrastructure loss, not just a single database restore
- Involves failing over to a secondary site/environment end-to-end
- Coordinates across teams (network, storage, application, DBA)
- Validates that DR documentation/runbooks are accurate and complete
- Measures actual recovery time against business-defined RTO/RPO
- Identifies gaps in automation, access, or dependencies not seen in routine tests
- Produces a formal after-action report with remediation items

**Q79. How do you handle backup/recovery planning for databases with different criticality tiers?**
- Classify databases by business criticality (Tier 1/2/3)
- Apply tighter RPO/RTO and more frequent testing to Tier 1 systems
- Align backup frequency and retention with each tier's requirements
- Prioritize DR resources and recovery order by tier during an incident
- Document tier classifications and review them periodically
- Communicate tier-based SLAs clearly to business stakeholders
- Validate that infrastructure (storage/network) supports tier commitments

**Q80. What do you do if a restore test reveals the recovery plan doesn't meet RTO?**
- Document the actual recovery time versus the target gap
- Analyze bottlenecks (I/O throughput, backup size, network transfer)
- Evaluate options: differential backups, faster storage, backup compression
- Consider Always On or log shipping for tighter RTO where justified
- Re-test after implementing improvements to confirm the gap is closed
- Communicate revised RTO expectations to stakeholders if still not achievable
- Update the recovery plan documentation with the improved process

---

## Section 17 — Storage, Archiving, Backup & Recovery Procedure Verification

**Q81. How do you verify that archiving procedures are functioning correctly?**
- Confirm archive jobs are running on schedule without failure
- Validate archived data is retrievable and matches source records
- Check that archived data is purged from production per retention rules only after successful archive confirmation
- Monitor archive storage capacity trends
- Periodically sample-test archived records for completeness/integrity
- Ensure archive jobs are logged and alertable on failure
- Review archiving procedures against current retention policy periodically

**Q82. How do you monitor storage growth to avoid capacity-related failures?**
- Track database and log file growth trends over time
- Set proactive alerts at defined storage utilization thresholds
- Review autogrowth event history for unexpected spikes
- Correlate growth trends with upcoming business/data volume changes
- Plan storage additions ahead of projected capacity exhaustion
- Report storage trends to infrastructure teams for procurement planning
- Validate that backup storage (NAS) growth is tracked separately from data storage

**Q83. What checks confirm backup and recovery procedures are functioning as designed, on an ongoing basis?**
- Daily review of backup job success/failure status across all databases
- Periodic automated or manual restore verification tests
- Backup file integrity checks (checksum validation)
- Confirming backup retention/cleanup jobs behave as configured
- Reviewing backup duration trends for early warning of degradation
- Validating alerting fires correctly on simulated failure scenarios
- Documenting verification results for audit evidence

**Q84. How do you handle storage-related incidents (e.g., disk-full events)?**
- Identify the affected drive/volume and immediate impact (data/log/tempdb)
- Free space quickly via log backups, temp file cleanup, or emergency growth
- Communicate impact/status to stakeholders during the incident
- Identify root cause (unexpected growth, failed cleanup job, etc.)
- Implement monitoring/alerting improvements to catch the trend earlier next time
- Document the incident and remediation for the knowledge base
- Review capacity planning assumptions in light of the incident

**Q85. What role does storage tiering play in backup/archive strategy?**
- Recent backups placed on fast, readily accessible storage for quick restores
- Older backups moved to lower-cost, higher-latency storage (cold/archive tier)
- Retention policy dictates when data moves between tiers
- Tiering reduces primary storage cost while meeting retention obligations
- Requires validation that archived-tier restores still meet acceptable RTO
- Automation should handle tier transitions rather than manual processes
- Tiering strategy should be documented in the backup/recovery SOP

---

## Section 18 — Capacity Planning

**Q86. How do you conduct capacity planning for SQL Server infrastructure?**
- Analyze historical growth trends for data, log, and tempdb storage
- Project future growth based on business initiatives and data volume trends
- Assess CPU and memory utilization trends against current allocation
- Evaluate I/O throughput headroom against projected workload growth
- Factor in upcoming projects/migrations that will affect resource needs
- Present findings and recommendations to infrastructure/budget stakeholders
- Revisit the capacity plan on a regular review cycle

**Q87. What metrics are most important when forecasting future storage needs?**
- Database and log file growth rate over recent history
- Backup storage growth trend (including retention policy impact)
- Index and tempdb growth patterns tied to workload changes
- Data archiving/purge effectiveness reducing active growth
- Planned application/data initiatives likely to accelerate growth
- Current available headroom versus lead time to provision more storage
- Seasonal or cyclical spikes in data volume

**Q88. How do you plan for compute (CPU/memory) capacity as workload grows?**
- Track baseline CPU/memory utilization trends over time
- Correlate utilization spikes with specific workloads or batch windows
- Identify headroom remaining before performance degradation is expected
- Evaluate whether query/index optimization can defer the need for more hardware
- Model expected impact of known upcoming workload additions
- Recommend scale-up/scale-out or cloud elasticity options as appropriate
- Validate licensing implications of any capacity changes

**Q89. How does capacity planning differ for cloud-hosted (AWS/Azure) SQL Server versus on-premises?**
- Cloud allows more elastic scaling but requires cost-awareness in planning
- On-premises requires longer procurement lead times, demanding earlier forecasting
- Cloud metrics (CloudWatch/Azure Monitor) supplement native SQL Server DMVs
- Reserved capacity/instance sizing decisions affect cost efficiency in cloud
- Storage autoscaling options differ (e.g., RDS storage autoscaling vs. manual on-prem expansion)
- DR/HA capacity considerations differ (multi-AZ, geo-redundancy options)
- Both require the same underlying workload trend analysis as a foundation

**Q90. What do you include in a capacity planning report for leadership?**
- Current utilization summary across CPU, memory, storage, and I/O
- Growth trend charts with projected exhaustion timelines
- Risk assessment for near-term capacity constraints
- Recommended actions with associated cost/lead-time estimates
- Impact of known upcoming projects on capacity needs
- Comparison against previous capacity plan projections (accuracy check)
- Clear timeline recommendation for when action is needed

---

## Section 19 — 24x7 Production Support & On-Call

**Q91. How do you prepare to be effective during on-call rotations?**
- Maintain up-to-date runbooks for common incident types
- Ensure remote access and required tool credentials are ready before the shift
- Review recent changes/deployments that could affect stability during the shift
- Confirm escalation contacts and paths are current
- Keep monitoring/alerting dashboards accessible from remote locations
- Review open tickets/known issues at shift handoff
- Test notification channels (phone/pager/email) before the rotation begins

**Q92. What's your approach to triaging a 2 a.m. production alert?**
- Assess severity and business impact immediately from available monitoring data
- Apply the relevant runbook for the specific alert type
- Determine if immediate mitigation is possible without full root-cause analysis
- Escalate to secondary on-call or subject matter experts if beyond scope
- Communicate status via the incident channel per the on-call protocol
- Stabilize the system first, then investigate root cause post-stabilization
- Document actions taken in real time for the post-incident review

**Q93. How do you balance minimizing downtime with proper root-cause investigation during an incident?**
- Prioritize service restoration first for high-impact production issues
- Capture diagnostic data (logs, plans, DMV snapshots) before applying a fix, if time allows
- Apply a known-safe mitigation rather than an untested fix under pressure
- Schedule deeper root-cause analysis after the immediate incident is resolved
- Avoid destructive troubleshooting steps that would erase diagnostic evidence
- Communicate a "resolved but investigating" status if root cause isn't yet confirmed
- Follow up with a permanent fix once root cause is understood

**Q94. What's included in an effective on-call handoff?**
- Summary of open incidents and their current status
- Any recent changes/deployments that could still be causing issues
- Known flaky alerts or false positives to be aware of
- Pending follow-up actions from resolved incidents
- Any planned maintenance during the upcoming shift
- Current system health/monitoring status snapshot
- Contact info for anyone actively engaged on an unresolved issue

**Q95. How do you reduce on-call burden and alert fatigue over time?**
- Tune alert thresholds to reduce false positives/noise
- Automate remediation for well-understood, recurring issues
- Conduct post-incident reviews to eliminate recurring root causes
- Improve documentation so on-call responders can resolve issues faster
- Track on-call incident volume/type trends to target improvement areas
- Cross-train team members to distribute on-call load fairly
- Advocate for proactive fixes over repeated reactive firefighting

---

## Section 20 — SDLC & General Practices

**Q96. How does the DBA function fit into the overall Software Development Life Cycle?**
- Participates in the design phase to review data model and performance implications
- Supports development with query/index guidance and code review
- Validates database changes during integration/testing phases
- Manages deployment of schema/data changes through controlled releases
- Monitors post-implementation performance and stability
- Maintains the database throughout the application's operational life
- Supports debugging and defect resolution across all SDLC phases

**Q97. How do you support integration and testing phases from a database perspective?**
- Ensure test environments have representative schema and data volumes
- Validate that database changes deploy cleanly via the same scripts used for Production
- Support performance/load testing with realistic query patterns
- Identify and resolve data-related defects found during testing
- Confirm rollback scripts work correctly before sign-off
- Coordinate with QA on database-related test case coverage
- Sign off on database readiness as part of release criteria

**Q98. What is your role during the implementation/deployment phase of a project?**
- Review and validate deployment scripts before execution
- Execute or oversee the database portion of the release during the maintenance window
- Monitor system health immediately following deployment
- Be available for rapid rollback if the deployment introduces issues
- Communicate deployment status to project stakeholders
- Update documentation to reflect the newly deployed state
- Conduct a brief post-deployment validation checklist

**Q99. How do you approach debugging a defect that spans application and database layers?**
- Reproduce the issue and isolate whether it's application logic or data/query behavior
- Collaborate with developers to trace the full request path
- Use profiling/tracing tools to pinpoint where the defect originates
- Validate data integrity assumptions the application code relies on
- Propose and test a fix at the layer where the root cause actually lives
- Verify the fix doesn't introduce regressions elsewhere
- Document the defect and resolution for future reference

**Q100. What toolset do you typically use across the SDLC for database work, and how are they applied?**
- SSMS for day-to-day development, administration, and querying
- SQL Profiler / Extended Events for performance troubleshooting
- SSIS for ETL development and data integration
- SSRS for report development and delivery
- SSAS for analytical/OLAP data structures where needed
- Import/Export Utility and BCP for bulk data movement
- Commvault for backup/recovery, Perfmon for OS-level monitoring, ARGENT Scheduler for batch orchestration, ERwin for data modeling, and MS VSS/TFS for source control and change tracking, with MS Visio for architecture/process diagrams

---
