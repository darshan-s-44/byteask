# Build 2026 × ByteAsk Open-Source Challenge Submission

**Participant / Team**: Darshan S (`darshan-s-44`)  
**Registered Email**: `darshan.cbe@gmail.com`  
**Target Repository**: [IvorySQL/IvorySQL](https://github.com/IvorySQL/IvorySQL) (★ 1,067+ Stars)  
**Target Issue**: [#2342: Build failure in DBMS_SCHEDULER due to incorrect placement of pg_noreturn](https://github.com/IvorySQL/IvorySQL/issues/2342)  
**Upstream Pull Request**: [IvorySQL/IvorySQL#2356](https://github.com/IvorySQL/IvorySQL/pull/2356)  

---

## 📌 Executive Summary

This repository documents the investigation, root-cause diagnosis, and patch implementation submitted for the **Build 2026 × ByteAsk Open-Source Challenge**.

The patch resolves a critical C / C23 build break in IvorySQL's Oracle compatibility module (`contrib/ivorysql_ora`), specifically within the `DBMS_SCHEDULER` built-in package declarations.

---

## 🔍 Problem Analysis & Root Cause

### 1. The Issue
When building IvorySQL with modern toolchains where `pg_noreturn` expands to the standard C23 attribute `[[noreturn]]` (defined in `src/include/c.h`), the compilation of `contrib/ivorysql_ora/src/ivorysql_ora.c` fails with compiler errors:

```log
In file included from src/ivorysql_ora.c:45:
src/builtin_packages/dbms_scheduler/dbms_scheduler.h:153:1: warning: 'noreturn' attribute ignored [-Wattributes]
  153 | extern PGDLLEXPORT pg_noreturn void SchedulerLauncherMain(Datum main_arg);
      | ^~~~~~
src/builtin_packages/dbms_scheduler/dbms_scheduler.h:153:32: error: expected identifier or '(' before 'void'
  153 | extern PGDLLEXPORT pg_noreturn void SchedulerLauncherMain(Datum main_arg);
      |                                ^~~~
src/builtin_packages/dbms_scheduler/dbms_scheduler.h:154:1: warning: 'noreturn' attribute ignored [-Wattributes]
  154 | extern PGDLLEXPORT pg_noreturn void SchedulerDatabaseWorkerMain(Datum main_arg);
      | ^~~~~~
src/builtin_packages/dbms_scheduler/dbms_scheduler.h:154:32: error: expected identifier or '(' before 'void'
  154 | extern PGDLLEXPORT pg_noreturn void SchedulerDatabaseWorkerMain(Datum main_arg);
      |                                ^~~~
src/builtin_packages/dbms_scheduler/dbms_scheduler.h:155:1: warning: 'noreturn' attribute ignored [-Wattributes]
  155 | extern PGDLLEXPORT pg_noreturn void SchedulerJobWorkerMain(Datum main_arg);
      | ^~~~~~
src/builtin_packages/dbms_scheduler/dbms_scheduler.h:155:32: error: expected identifier or '(' before 'void'
  155 | extern PGDLLEXPORT pg_noreturn void SchedulerJobWorkerMain(Datum main_arg);
      |                                ^~~~
make[1]: *** [<builtin>: src/ivorysql_ora.o] Error 1
make: *** [GNUmakefile:11: all-contrib/ivorysql_ora-recurse] Error 2
```

### 2. Technical Root Cause
In ISO C (C23 standard), function attribute specifiers such as `[[noreturn]]` apply to the declaration and must be placed at the beginning of the declaration (or directly preceding the identifier in a declarator). 

In `dbms_scheduler.h`, the macro `pg_noreturn` was erroneously placed **after** `extern PGDLLEXPORT` and before `void`:
```c
extern PGDLLEXPORT pg_noreturn void SchedulerLauncherMain(Datum main_arg);
```
Because `extern` and `PGDLLEXPORT` precede `pg_noreturn`, modern compilers reject the attribute token order as a misplaced attribute and syntax error.

### 3. Standards Compliance
Across upstream PostgreSQL and the rest of the IvorySQL codebase (e.g. in `src/include/utils/elog.h`, `src/include/bootstrap/bootstrap.h`, `src/include/postmaster/postmaster.h`), `pg_noreturn` is consistently positioned **at the start of the declaration** before `extern`:

```c
/* Standard PostgreSQL convention */
pg_noreturn extern void PostmasterMain(int argc, char *argv[]);
pg_noreturn extern void ExceptionalCondition(const char *conditionName, ...);
```

---

## 🛠️ The Fix (Minimal Diff)

**File modified**: `contrib/ivorysql_ora/src/builtin_packages/dbms_scheduler/dbms_scheduler.h`

```diff
diff --git a/contrib/ivorysql_ora/src/builtin_packages/dbms_scheduler/dbms_scheduler.h b/contrib/ivorysql_ora/src/builtin_packages/dbms_scheduler/dbms_scheduler.h
index 8e4dfb1..4cb0912 100644
--- a/contrib/ivorysql_ora/src/builtin_packages/dbms_scheduler/dbms_scheduler.h
+++ b/contrib/ivorysql_ora/src/builtin_packages/dbms_scheduler/dbms_scheduler.h
@@ -150,9 +150,9 @@ extern void SchedulerLauncherRegister(void);
 
 /* bgworker entry points, looked up by name in the shared library */
-extern PGDLLEXPORT pg_noreturn void SchedulerLauncherMain(Datum main_arg);
-extern PGDLLEXPORT pg_noreturn void SchedulerDatabaseWorkerMain(Datum main_arg);
-extern PGDLLEXPORT pg_noreturn void SchedulerJobWorkerMain(Datum main_arg);
+pg_noreturn extern PGDLLEXPORT void SchedulerLauncherMain(Datum main_arg);
+pg_noreturn extern PGDLLEXPORT void SchedulerDatabaseWorkerMain(Datum main_arg);
+pg_noreturn extern PGDLLEXPORT void SchedulerJobWorkerMain(Datum main_arg);
 
 /* Set when the launcher was registered at shared_preload time */
 extern bool scheduler_launcher_registered;
```

---

## ✅ Verification & Impact

1. **Zero Side Effects**: Changing the specifier order has no impact on runtime semantics or symbol linkage (`PGDLLEXPORT` is preserved).
2. **Compiler Compatibility**: Resolves both GCC and Clang `-Wattributes` warnings and eliminates C23 parse errors.
3. **Coding Convention Alignment**: Conforms with PostgreSQL coding standards defined in `CONTRIBUTING.md`.

---

## 📋 Challenge Submission Details

* **Registered Email**: `darshan.cbe@gmail.com`
* **Form URL**: [Google Form](https://forms.gle/5KTw1xEiahChgtvHA)
* **What the PR does**:
  > Fixed C23 build failure in DBMS_SCHEDULER (Issue #2342) in IvorySQL/IvorySQL by moving pg_noreturn before extern PGDLLEXPORT for background worker entry point declarations in dbms_scheduler.h to comply with C23 attribute syntax and codebase conventions.
