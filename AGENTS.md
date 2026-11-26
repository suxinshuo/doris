# AGENTS.md

Apache Doris: Java/Maven FE, C++/CMake BE, React UI, Thrift/Proto codegen, and a Groovy regression harness. This checkout is a 3.0.6.2 FE hotfix branch; production deploys `doris-fe.jar`.

## Build

- Single entrypoint: `./build.sh`. With no args it builds FE + BE + Cloud + Broker + Spark DPP + Hive UDF + BE Java extensions (very slow); always pass components explicitly.
- JDK 17 is mandatory for building and for running the FE (`env.sh` check; `bin/start_fe.sh` refuses other versions). On macOS `env.sh` regenerates `custom_env_mac.sh` from Homebrew. `build.sh` initializes the BE submodules (`apache-orc`, `clucene`) and builds thirdparty only if `thirdparty/installed/lib/libbacktrace.a` is missing.
- FE-only build (no UI, Spark DPP, Hive UDF, BE extensions, or checkstyle):

  ```bash
  DISABLE_BUILD_UI=ON DISABLE_BUILD_SPARK_DPP=ON DISABLE_BUILD_HIVE_UDF=ON \
  DISABLE_BE_JAVA_EXTENSIONS=ON DISABLE_JAVA_CHECK_STYLE=ON \
  MVN_OPT=-llr ./build.sh --fe --clean
  ```

  `DISABLE_BUILD_UI=ON` is only safe when `fe/fe-core/src/main/resources/static/` is populated. That directory is gitignored; on a fresh clone run `./build.sh --fe` once without the flag to build the UI.
- Maven gotcha: the corporate Nexus mirror in `~/.m2/settings.xml` lacks some old plugins, e.g. `net.sourceforge.czt.dev:cup-maven-plugin:1.6-cdh`. If the artifact exists in the local repo but Maven still tries the network, add `MVN_OPT=-llr` to `build.sh` (or `-llr` to `mvn`). This is required on this workstation.
- The real version comes from `gensrc/script/gen_build_version.sh`; `fe/pom.xml`'s `revision=1.2-SNAPSHOT` is not the Doris release version.
- `--clean` wipes `gensrc/build` and the FE targets, then regenerates Thrift/parquet/`Version.java`. Use it after switching commits: `Version.java` embeds the git HEAD hash, and an incremental build can keep an old hash.
- Generated Java under `fe/fe-common/src/main/java/org/apache/doris/{thrift,parquet}` is gitignored. Edit `gensrc/thrift`, `gensrc/proto`, or `fe/fe-core/src/main/cup|jflex` instead of the generated files.
- Artifacts: `fe/fe-core/target/doris-fe.jar` (same file copied to `output/fe/lib/doris-fe.jar`); dependencies are copied to `output/fe/lib`. `output/fe/conf` is copied from the local `conf/`, which may contain local edits, so never deploy the whole `output/` tree.

## Tests

- FE unit tests: `./run-fe-ut.sh` for all, or `./run-fe-ut.sh --run org.apache.doris.SomeTest#method`. The script regenerates sources and runs Maven with `-Dtest=...`. Fork/parallel defaults to 1 (`FE_UT_PARALLEL`); raise it only deliberately, since FE tests are not all isolation-safe.
- BE unit tests: `./run-be-ut.sh --run --filter='FooTest.*'`. The first run compiles BE. Test files must end in `_test` and be registered in `be/test/CMakeLists.txt`.
- Regression tests: `./run-regression-test.sh --run -s <suite>` requires a running cluster (defaults in `regression-test/conf/regression-conf.groovy` point at `127.0.0.1:9030`). Expected outputs live under `regression-test/data/`.
- Style: Java `cd fe && mvn clean checkstyle:check` (what CI runs); C++ `build-support/check-format.sh` (clang-format 16). New files need the Apache license header (`.licenserc.yaml`). PR titles must match `[type](scope) description` (CI title checker).

## FE hotfix deploy

- If only `fe/fe-core` changed, replace only `lib/doris-fe.jar` on each FE; `fe-common` and dependency jars are unchanged. Back up the old jar first.
- Rolling order: followers/observers first, master last, one FE at a time to keep quorum. `./bin/stop_fe.sh --grace` sends SIGTERM; the default is SIGKILL.
- Verify with `SHOW FRONTENDS\G`: `Alive=true`, and `Version` is `<DORIS_BUILD_VERSION>-<short git hash>` (e.g. `doris-3.0.6.2-rc01-11b2ec5a00e`). Roll back by restoring the backed-up jar and restarting.
- Never connect an agent to the production cluster or database; build the artifact and hand the commands to the user.

## Layout

- `fe/fe-core`: main FE (`org.apache.doris.DorisFE`), Nereids optimizer in `nereids/`, legacy planner in `planner/`, metadata in `catalog/`, Ranger authorizer in `catalog/authorizer/ranger/` (the Hive service-def uses lower-case access types; see `getRowFilterAndDataMaskAccessType()`).
- `fe/fe-common`: shared code plus generated Thrift; `fe/spark-dpp`, `fe/hive-udf`, `fe/be-java-extensions` are optional modules.
- `be/src`, `be/test`; `cloud/`; `gensrc/{thrift,proto}` plus `gensrc/script/gen_build_version.sh`; `regression-test/suites` and `regression-test/data`.
- `ui/`: React + antd, built into `fe-core` resources by `build.sh`.

## Working agreements

- Ask before changing business code; this branch ships to production.
- Stage only files belonging to the task and commit completed tasks separately. Do not add AI/assistant attribution to commit messages.
- The working tree contains local edits (`env.sh`, `conf/fe.conf`, `fe/.idea/vcs.xml`) and untracked BE CMake artifacts. Do not revert, delete, or commit them unless asked.
