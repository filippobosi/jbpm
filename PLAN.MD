# Jakarta Migration Execution Plan

## Purpose

This file is a task-oriented execution plan that can be handed to an AI agent working on the `jbpm` repository.

The goal is to migrate the project toward:

- `jakarta.*` APIs instead of `javax.*`
- compatibility with `JDK 17`
- incremental validation by module cluster

This plan is intentionally operational. Each task includes:

- objective
- scope
- concrete actions
- expected output
- done criteria
- dependencies

The agent should work incrementally and avoid a repo-wide big bang migration.

## Global Execution Rules

- Do not migrate the entire repository in one step.
- Work by module cluster.
- Prefer small, reviewable commits or patches.
- Apply OpenRewrite first for mechanical changes, then do manual cleanup.
- After each cluster migration, run compilation and focused tests before moving on.
- Treat OSGi metadata, XML descriptors, and container tests as explicit work items.
- Do not start with `jbpm-container-test`, `jbpm-distribution`, or `jbpm-installer`.
- Do not assume that a successful compile means Jakarta compatibility is complete.

## Global Technical Target

- Jakarta target: `Jakarta EE 9` namespace migration first
- Java target: `JDK 17`
- Strategy: migrate runtime code first, then descriptors/configuration, then test/container layers

## Suggested OpenRewrite Recipes

Primary recipes:

- `org.openrewrite.java.migrate.jakarta.MigrateToJakartaEe9`
- `org.openrewrite.java.migrate.jakarta.JavaxPersistenceToJakartaPersistence`
- `org.openrewrite.java.migrate.jakarta.JavaxTransactionToJakartaTransaction`
- `org.openrewrite.java.migrate.jakarta.JavaxJmsToJakartaJms`
- `org.openrewrite.java.migrate.jakarta.JavaxInjectToJakartaInject`
- `org.openrewrite.java.migrate.jakarta.JavaxAnnotationToJakartaAnnotation`
- `org.openrewrite.java.migrate.jakarta.JavaxXmlBindToJakartaXmlBind`
- `org.openrewrite.java.migrate.jakarta.JavaxMailToJakartaMail`
- `org.openrewrite.java.migrate.jakarta.JavaxServletToJakartaServlet`
- `org.openrewrite.java.migrate.jakarta.JavaxWsToJakartaWs`
- `org.openrewrite.java.migrate.jakarta.JavaxEjbToJakartaEjb`
- `org.openrewrite.java.migrate.jakarta.JavaxEnterpriseToJakartaEnterprise`

Later-stage recipe:

- `org.openrewrite.java.migrate.UpgradeToJava17`

## Module Waves

mvn -Prewrite org.openrewrite.maven:rewrite-maven-plugin:dryRun \
-Drewrite.configLocation=rewrite.yml \
-Drewrite.activeRecipes=org.jbpm.rewrite.JakartaWave1 \
-pl jbpm-persistence,jbpm-query-jpa,jbpm-audit,jbpm-runtime-manager

### Wave 1

- `jbpm-persistence`
- `jbpm-query-jpa`
- `jbpm-audit`
- `jbpm-runtime-manager`

### Wave 2

- `jbpm-human-task`
- `jbpm-services`
- `jbpm-case-mgmt`

### Wave 3

- `jbpm-workitems`
- `jbpm-xes`
- `jbpm-document`
- `jbpm-event-emitters`
- `jbpm-flow`
- `jbpm-bpmn2`
- `jbpm-flow-builder`

### Wave 4

- `jbpm-test`
- `jbpm-test-util`
- `jbpm-test-coverage`

### Wave 5

- `jbpm-container-test`
- `jbpm-distribution`
- `jbpm-installer`
- `jbpm-examples`

## Task 0 - Establish Build Baseline

### Objective

Record the current build status before changing anything.

### Scope

Root project and Wave 1 modules.

### Actions

- Capture `java -version` and `mvn -v`.
- Confirm current effective Java runtime used by Maven.
- Run a baseline compile for Wave 1 modules only.
- Record failing modules, failing tests, and environment-sensitive failures.

### Expected Output

- short baseline report in Markdown
- list of modules that already fail before migration

### Done Criteria

- baseline report exists
- compile/test status for Wave 1 is recorded

### Dependencies

- none

## Task 1 - Add OpenRewrite Configuration

### Objective

Prepare a repeatable rewrite entry point.

### Scope

Root `pom.xml` and shared build configuration.

### Actions

- add or configure the OpenRewrite Maven plugin
- define a reusable recipe set for Jakarta migration
- keep the configuration isolated in a profile if possible
- make the plugin runnable without changing all modules at once

### Expected Output

- OpenRewrite plugin configuration in Maven
- documented command to run recipes on selected modules

### Done Criteria

- `rewrite:dryRun` or equivalent can execute successfully on at least one target module

### Dependencies

- Task 0

## Task 2 - Create Dependency Convergence Inventory

### Objective

Identify conflicting or residual `javax` dependencies before mass rewrite.

### Scope

All Maven modules, with priority on Wave 1 and Wave 2.

### Actions

- scan all `pom.xml` files for:
  - `javax.*`
  - `org.jboss.spec.javax.*`
  - `org.hibernate.javax.persistence`
  - mixed `javax` and `jakarta` APIs in the same module
- classify findings as:
  - runtime blocker
  - test-only blocker
  - packaging/configuration blocker

### Expected Output

- inventory table: `module | dependency | severity | action`

### Done Criteria

- all Wave 1 and Wave 2 modules are classified

### Dependencies

- Task 0

## Task 3 - Migrate Wave 1 With OpenRewrite

### Objective

Apply the first mechanical migration on the core persistence/runtime cluster.

### Scope

- `jbpm-persistence`
- `jbpm-query-jpa`
- `jbpm-audit`
- `jbpm-runtime-manager`

### Actions

- run Jakarta rewrite recipes on Wave 1 modules only
- inspect generated diffs
- keep changes limited to Wave 1
- do not touch container-test modules

### Expected Output

- mechanical import and dependency migration for Wave 1

### Done Criteria

- code compiles or reaches a smaller, well-understood failure set
- no uncontrolled rewrite spillover outside Wave 1

### Dependencies

- Task 1
- Task 2

## Task 4 - Manually Fix Wave 1 Compile Errors

### Objective

Resolve the non-mechanical issues left after OpenRewrite.

### Scope

Wave 1 modules only.

### Actions

- fix leftover imports not rewritten correctly
- resolve API mismatches and signature changes
- fix JAXB-related breakages
- resolve JMS/JPA/transaction references still bound to `javax`
- correct code that compiles against mixed classpaths

### Expected Output

- Wave 1 source compiles cleanly

### Done Criteria

- Wave 1 `src/main` compiles without `javax` runtime references in migrated areas

### Dependencies

- Task 3

## Task 5 - Fix Wave 1 OSGi Metadata And Packaging

### Objective

Align packaging metadata with migrated code.

### Scope

Wave 1 POMs and bundle metadata.

### Actions

- inspect `maven-bundle-plugin` configuration
- update `Import-Package` and related package lists from `javax.*` to `jakarta.*`
- remove obsolete imports where possible
- verify bundle metadata stays coherent with the new dependency graph

### Expected Output

- corrected OSGi metadata for Wave 1

### Done Criteria

- no Wave 1 bundle metadata still hardcodes old `javax` imports where Jakarta is now required

### Dependencies

- Task 4

## Task 6 - Fix Wave 1 XML And Descriptor References

### Objective

Remove legacy namespace and property references outside Java code.

### Scope

Wave 1 resources and related test resources.

### Actions

- inspect and migrate:
  - `persistence.xml`
  - `kmodule.xml`
  - other XML descriptors and resource files
- replace `javax.persistence.*` property names where needed
- flag any descriptor whose migration is unclear for manual review

### Expected Output

- updated descriptors/resources for Wave 1

### Done Criteria

- no known Wave 1 configuration file still blocks Jakarta runtime due to legacy namespace/property usage

### Dependencies

- Task 4

## Task 7 - Validate Wave 1

### Objective

Prove that the first migration wave is stable enough to continue.

### Scope

Wave 1 only.

### Actions

- run compile on Wave 1
- run focused unit/integration tests for Wave 1
- record remaining failures
- separate:
  - product regressions
  - test fragility
  - locale/environment issues

### Expected Output

- validation report with pass/fail status

### Done Criteria

- Wave 1 compile succeeds
- remaining failures are understood and documented

### Dependencies

- Task 5
- Task 6

## Task 8 - Migrate Wave 2 With OpenRewrite

### Objective

Apply the same migration approach to the largest application layer modules.

### Scope

- `jbpm-human-task`
- `jbpm-services`
- `jbpm-case-mgmt`

### Actions

- run Jakarta rewrite recipes on Wave 2
- keep the work isolated from Wave 3+ modules
- prioritize:
  - JPA
  - JAXB
  - JMS
  - CDI
  - EJB

### Expected Output

- mechanical migration of Wave 2

### Done Criteria

- Wave 2 rewrite completes and generated failures are triaged

### Dependencies

- Task 7

## Task 9 - Manually Fix Wave 2

### Objective

Resolve migration fallout in the services and human-task layer.

### Scope

Wave 2 only.

### Actions

- fix remaining `javax` imports and type references
- update JMS listeners/producers/consumers
- update CDI/EJB integration points
- resolve persistence and serialization issues
- fix obvious test assumptions broken by migration

### Expected Output

- Wave 2 compiles cleanly or with a small documented residual failure set

### Done Criteria

- runtime code in Wave 2 is migrated to Jakarta-compatible APIs

### Dependencies

- Task 8

## Task 10 - Fix Wave 2 Packaging, XML, And Validation

### Objective

Bring Wave 2 to the same quality gate as Wave 1.

### Scope

Wave 2 metadata/resources/tests.

### Actions

- update OSGi metadata and package imports
- migrate XML descriptors and configuration
- run focused tests
- document unresolved failures and classify them

### Expected Output

- validation report for Wave 2

### Done Criteria

- Wave 2 compile succeeds
- major blockers are either fixed or formally listed

### Dependencies

- Task 9

## Task 11 - Migrate Wave 3

### Objective

Migrate secondary runtime and integration modules.

### Scope

- `jbpm-workitems`
- `jbpm-xes`
- `jbpm-document`
- `jbpm-event-emitters`
- `jbpm-flow`
- `jbpm-bpmn2`
- `jbpm-flow-builder`

### Actions

- apply OpenRewrite
- fix manual issues
- focus especially on:
  - JAXB
  - servlet/rest/webservice APIs
  - mail/activation
  - BPMN parsing/serialization

### Expected Output

- Wave 3 migrated and validated

### Done Criteria

- Wave 3 compile succeeds
- no unreviewed `javax` runtime references remain in migrated modules

### Dependencies

- Task 10

## Task 12 - Stabilize Test Modules Under JDK 17

### Objective

Handle test-only breakages after runtime migration.

### Scope

- `jbpm-test`
- `jbpm-test-util`
- `jbpm-test-coverage`

### Actions

- migrate remaining test code from `javax` to `jakarta`
- fix locale-sensitive assertions
- fix parser-message assertions that expect English-only text
- review tests using `SecurityManager`, `Byteman`, or JVM-sensitive setup
- identify tests incompatible with modern JVM behavior and isolate them

### Expected Output

- stable test support modules

### Done Criteria

- test modules compile
- high-value test suites run on JDK 17 with known exceptions documented

### Dependencies

- Task 11

## Task 13 - Migrate Container And Environment-Specific Modules

### Objective

Address the highest-risk, lowest-predictability integration layer last.

### Scope

- `jbpm-container-test`
- `jbpm-distribution`
- `jbpm-installer`
- `jbpm-examples`

### Actions

- migrate remaining `javax` dependencies and descriptors
- update WildFly/JBoss/Arquillian related configuration
- review spec dependencies and replace legacy variants where possible
- validate in-container behavior where practical

### Expected Output

- migrated environment-facing modules

### Done Criteria

- each module has either:
  - successful migration validation
  - or a clearly documented unresolved blocker

### Dependencies

- Task 12

## Task 14 - Apply Java 17 Upgrade Cleanup

### Objective

Finalize Java 17 compatibility after Jakarta migration is mostly stable.

### Scope

All migrated modules.

### Actions

- apply `UpgradeToJava17` selectively
- fix code or tests affected by JDK 17 behavior
- review deprecated or removed mechanisms
- pay special attention to:
  - `SecurityManager`-related tests
  - reflective access assumptions
  - JAXB/provider behavior

### Expected Output

- Java 17 cleanup changes

### Done Criteria

- migrated modules compile and run under JDK 17 with documented residual gaps only

### Dependencies

- Task 13

## Task 15 - Add Regression Guards

### Objective

Prevent backsliding after the migration.

### Scope

Root build and CI guardrails.

### Actions

- add checks to detect new `javax.*` imports in migrated modules
- add checks for residual legacy Maven dependencies where appropriate
- document known exceptions explicitly
- update project documentation

### Expected Output

- CI or build guardrails
- short migration notes for maintainers

### Done Criteria

- new regressions toward legacy APIs are automatically detected

### Dependencies

- Task 14

## Success Criteria

The migration can be considered successful for the agreed scope when:

- migrated runtime modules no longer rely on legacy `javax.*` APIs
- Maven dependencies are coherent with Jakarta
- OSGi/package metadata is aligned with the new APIs
- XML/configuration descriptors are updated
- targeted tests pass on JDK 17
- residual blockers are explicitly documented, not hidden

## Minimum Deliverables Expected From The Agent

- updated Maven/OpenRewrite configuration
- migrated code per wave
- validation notes after each wave
- blocker list with severity and module ownership
- final residual gap report

## Execution Notes For The Agent

- Prefer completing one wave fully before starting the next.
- If a task reveals a blocker outside its wave, document it but do not broaden scope automatically.
- When tests fail, distinguish product issue from environment issue from fragile assertion.
- Keep changes grouped by module wave for easier review.


## note

mvn -Prewrite org.openrewrite.maven:rewrite-maven-plugin:dryRun \
-Drewrite.configLocation=rewrite.yml \
-Drewrite.activeRecipes=org.jbpm.rewrite.JakartaWave1 \
-pl jbpm-persistence,jbpm-query-jpa,jbpm-audit,jbpm-runtime-manager