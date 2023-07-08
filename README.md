# WeTune
WeTune is a rule generator that can automatically
discover new query rewrite rules for SQL query optimization.

## Publications
Zhaoguo Wang, Zhou Zhou, Yicun Yang, Haoran Ding, Gansen Hu,
Ding Ding, Chuzhe Tang, Haibo Chen, and Jinyang Li. 2022.
[WeTune: Automatic Discovery and Verification of Query Rewrite Rules.](https://dl.acm.org/doi/10.1145/3514221.3526125)
In Proceedings of the 2022 International Conference on Management of Data (SIGMOD ’22).

## CodeBase
This codebase includes the source code and the testing scripts in the paper
*Automatic Discovery and Verification of Query Rewrite Rules*

```shell
.
|-- click-to-run/    # Click-to-run scripts for WeTune.
|-- docker/          # One-click-reproduce-all scripts for WeTune's reproducibility (use Docker).
|-- lib/             # Required external library.
|-- common/          # Common utilities.
|-- spes/            # Integrated SPES-based verifier.
|-- sql/             # Data structures of SQL AST and query plan.
|-- stmt/            # Manager of queries from open-source applications.
|-- superopt/        # Core algorithm of WeTune.
    |-- fragment/    # Plan template enumeration.
    |-- constraint/  # Constraint enumeration.
    |-- uexpr/       # U-expression.
    |-- logic/       # SMT-based verifier.
    |-- optimizer/   # Rewriter.
|-- testbed/         # Evaluation framework.
|-- wtune_data/      # Data input/output directory.
    |-- schemas/     # Schemas of applications.
    |-- schemas_mssql/ # Schemas of applications for SQL Server
    |-- wtune.db     # Sqlite DB storing the persistent statistics
```

## Reproducibility
The scripts provided in `{projRootDir}/docker` are used for reproducibility.
The requirements for the machine are listed as follows:
* OS: Linux
* Apps: Docker, bash
* Disk space: at least 750 GiB disk space for the docker container *(most of which is used for storing tables when evaluating large workloads in SQL Server)*

Here are the steps to follow:
1. Download the repository of WeTune to your machine.
2. Enter the directory `{projRootDir}/docker`.
3. Update `HOST_DUMP_PATH`, `HOST_MSSQL_PATH` in the `repro.sh` script, so they point to the appropriate paths on the host machine.
   We suggest 350GiB (at least 250GiB) for `HOST_DUMP_PATH` and 650GiB (at least 600GiB) for `HOST_MSSQL_PATH`
4. Execute `repro.sh` to start the experiments.
5. Execute `repro-issue.sh` to get the rewriting result of 50 GitHub issues mentioned in the paper.

When the scripts finish, the result will be copied to the host machine under the directory
`{projRootDir}/docker/result_from_docker` and `{projRootDir}/docker/issue_result_from_docker`.
The directory structures are as follows:

```
./docker/result_from_docker/
├── calcite
│   └── result
│       ├── 1_query.tsv
│       ├── 1_rules.tsv
│       ├── 2_query.tsv
│       └── 2_trace.tsv
├── profile
│   └── result                            # the profiling results of four different workloads
│       ├── base
│       ├── large
│       ├── large_zipf
│       └── zipf
├── profile_calcite
│   └── result                            # the profiling result of four calcite
│       ├── base
│       ├── large
│       ├── large_zipf
│       └── zipf
├── rewrite
│   └── result
│       ├── 1_query.tsv                   # all rewritten queries
│       ├── 1_rules.tsv                   # rules used to rewrite queries in 1_query.tsv
│       ├── 2_query.tsv                   # the rewritten queries picked with the minimal cost by asking the database's cost
│       └── 2_trace.tsv                   # used rules of each rewritten query in 2_query.tsv
├── viewall
│   └── result
│       ├── optimizationInfo.tsv          # optimization info of queries
│       └── usefulRules.tsv               # all the used rules in the process of optimization
├── viewall_calcite
│   └── result
│       ├── statistic.tsv                 # optimization info of Calcite test suite
│       └── wetuneSurpassesCalcite.tsv    # queries WeTune achieves a better performance of rewriting than Calcite itself 
├── viewall_statistics
│    └── result
│       ├── base.tsv
│       ├── large.tsv
│       ├── large_zipf.tsv
│       ├── statistics                    # the statistics collected for the four workloads for the third checking point
│       └── zipf.tsv
├── enumeration
│        └── run_*                        # discovered rules so far ( * here is a timestamp)
│
└── rules.txt                             # the resulting rules that have been reduced
```

```
./docker/issue_result_from_docker/
└── issues
    ├── calcite_rewrites                                    # rewriting result of Apache Calcite
    ├── issues                                              # raw information of issues
    └── result                                              # the profiling results of four different workloads
        ├── {appName}#{issueId}                             # rewriting result of one certain issue identified by {appName}#{issueId}
        └── ...                                # raw information of issues
```

Notes: The two containers, `wetune` and `mssql`, communicate through port 1433 of the host machine.
So if you have any process using port 1433, you should stop it to free the port.


## Environment Setup

### Requirements

* Ubuntu 20.04 (or below)
* Java 17
* Gradle 7.3.3
* z3 4.8.9  *(SMT solver)*
* antlr 4.8  *(Generate tokenizer and parser for SQL AST)*
* Microsoft SQL Server 2019 *(Evaluate the usefulness of rules)*

z3 and antlr library have been put in `lib/` off-the-shelf.
You can follow the steps of the script below to set up your environment.

```shell
click-to-run/environment-setup.sh
```

### Compilation

```shell
gradle compileJava
```

## WeTune Workflow

This section gives the instruction of the whole workflow of WeTune, including

1. rule discovery
2. rewriting queries using rules
3. pick useful rules by evaluating the rewritings.

The whole procedure typically takes several days (mainly for rule discovery).

If you are particularly interested in how WeTune works, please refer to the section [Run Example](#run-examples), which
gives instructions of running a few individual examples of the enumeration and equivalence proof. The detailed output
will help understand the internal of WeTune.

### Discover Rules

```shell
# Launch background processes to run rule discovery
click-to-run/discover-rules.sh 

# After the all processes finished:
click-to-run/collect-rules.sh && click-to-run/reduce-rules.sh 

# Check progress:
click-to-run/show-progress.sh

# Use this to terminate all process
click-to-run/stop-discovery.sh
```

The first commands launches many processes running in the background.

The procedure will consume all CPUs and take a long time (~3600 CPU hours) to finish. The discovered rules so far can be
found in `wtune_data/enumeration/run_*/success.txt` (`*` here is a timestamp).

The second commands aggregates `wtune_data/enumeration/run_*/succcess.txt` and reduce the rules (Section 7 in paper).
The resulting rules can be found in `wtune_data/rules/rules.txt`.

The third are used to check the progress. The fourth can terminate all background tasks launched by the first command.

> **Why multi-process instead of multi-thread?**
>
> z3 incurs high inter-thread lock contention. The script uses multi-process instead of multi-thread to mitigate this problem.

> Since the rule discovery takes a substantial time, we have provided enumerated rules
> in `wtune_data/prepared/rules.txt`

### Rewrite Queries Using Discovered Rules

```shell
click-to-run/rewrite-queries.sh [-R <path/to/rules>]
```

This script uses the rules in `<path/to/rules>` to rewrite 8000+ web application queries.

* `-R`: path to rule file, rooted by `wtune_data/`. Default: `wtune_data/rules/rules.txt` if exists,
  otherwise `wtune_data/prepared/rules.txt`.

The rewritten queries can be found in `wtune_data/rewrite/result/1_query.tsv` and the used rules is stored in `wtune_data/rewrite/result/1_rules.tsv`.

### Evaluate the Rewritings

```shell
# Estimate the cost of rewritten queries and pick one with the minimal cost
# The prepared `base` workload is used to estimate the queries' cost
click-to-run/prepare-workload.sh -tag base
click-to-run/estimate-cost.sh

# Profile the performance of rewritten queries
# If you profile on the `base` workload type, no need to run `prepare-workload.sh` again
click-to-run/prepare-workload.sh [-tag <workload_type>]

click-to-run/profile-cost.sh [-tag <workload_type>]
```

These scripts pick the optimized queries and profile them using Sql Server database.

> For database connection parameters at line 8 ~ 11 of
> `click-to-run/make-db.sh`, line 71 ~ 74 of
> `click-to-run/generate-date.sh` (these 2 scripts are invoked by `click-to-run/prepare-workload.sh`),
> and the function `sqlserverProps` at line 58 of file
> `common/src/main/java/wtune/common/datasource/DbSupport.java`,
> you can change them according to your evaluation environment.**

Use `click-to-run/prepare-workload.sh` to prepare profiling workload data in Sql Server database.
It creates databases and corresponding schemas in Sql Server, then generate and import data to Sql Server.
Dumped data files can be found in directory `wtune_data/dump/`.

`click-to-run/estimate-cost.sh` takes previously generated file `wtune_data/rewrite/result/1_query.tsv` as input and
pick one rewritten query with the minimal cost by asking the database's cost model. The result will be stored
in `wtune_data/rewrite/result/2_query.tsv`. Used rules of each rewritten query (which are considered as useful rules) 
will be stored in `wtune_data/rewrite/result/2_trace.tsv`, represented as a sequence of rule ids separated by commas.
The id of each rule is exactly its line number (start from 1) of the file containing all the rules. 
And `click-to-run/profile-cost.sh` evaluates the optimized queries. The output file is in `wtune_data/profile/result/` by
default.

* `-tag`: specifies workload type. Default type is `base`. See details below.

#### Workload types
In the paper, we evaluate query performance on 4 different workload types:

| Workload type | # of rows | Data distribution |
|---------------|-----------|-------------------|
| base          | 10 k      | uniform           |
| zipf          | 10 k      | zipfian           |
| large         | 1 M       | uniform           |
| large_zipf    | 1 M       | zipfian           |

If you would like to evaluate on different type of workload, you
can set `-tag` option to the scripts.

For example, to evaluate queries on workload type of `zipf`, run:
```shell 
click-to-run/prepare-workload.sh -tag zipf 
click-to-run/profile-cost.sh -tag zipf 
```
The profiling result is actually stored in file `wtune_data/profile/result/{workload_type}` by default.

### View results
Finally, you can run:
```shell
click-to-run/view-all.sh
```
to view rewriting and profiling results together.
The resulting files are stored in `wtune_data/viewall/result/` by default.
```shell
|-- viewall
    |-- result
        |-- optimizationInfo.tsv       # Rewritten queries that are effective
        |-- usefulRules.tsv   # Rules used in the effective rewritten queries 
```

## SPES Verifier
WeTune has integrated an existing verifier called SPES
(Zhou et al. SPES: A Two-Stage Query Equivalence Verifier. VLDB '20.)
to improve the ability of discovering rules. The workflow of using SPES to discover
rules, rewrite and evaluate queries is similar to WeTune Workflow section.

### Discover Rules

```shell
# Launch background processes to run rule discovery
click-to-run/discover-rules-continuous.sh -spes

# After the all processes finished:
click-to-run/collect-rules.sh && click-to-run/reduce-rules.sh -spes

# Check progress:
click-to-run/show-progress.sh

# Use this to terminate all process
click-to-run/stop-discovery.sh
```

The first commands use SPES verifier to prove rule correctness. We use
`click-to-run/discover-rules-continuous.sh -spes` instead when using SPES verifier
since sometimes the processes may be unexpectedly shut down in SPES's
current implementation, which remains to be fixed in the future work.

The discovered rules so far can also be found in
`wtune_data/enumeration/run_*/success.txt` (`*` here is a timestamp).

The second commands aggregates `wtune_data/enumeration/run_*/succcess.txt`
and reduce the rules (Section 7 in paper).
The resulting rules can be found in `wtune_data/rules/rules.spes.txt`.

The third are used to check the progress. The fourth can terminate all background tasks launched by the first command.

> For the simplicity of demonstration, we separate the enumeration using the built-in and SPES verifier.

> We have also provided rules enumerated by SPES verifier in
> `wtune_data/prepared/rules.spes.txt`

### Rewrite Queries Using Discovered Rules
```shell
click-to-run/rewrite-queries.sh -spes [-R <path/to/rules>]
```

This script uses the rules in `<path/to/rules>` to rewrite queries.

* `-R`: path to rule file, rooted by `wtune_data/`. Default: `wtune_data/rules/rules.spes.txt` if exists,
  otherwise `wtune_data/prepared/rules.spes.txt`.

The rewritten queries can be found in `wtune_data/rewrite/result_spes/1_query.tsv` and the used rules is stored in `wtune_data/rewrite/result_spes/1_rules.tsv`.

### Evaluate the Rewritings

```shell
# Estimate the cost of rewritten queries and pick one with the minimal cost
# The prepared `base` workload is used to estimate the queries' cost
click-to-run/prepare-workload.sh -tag base
click-to-run/estimate-cost.sh -spes

# Profile the performance of rewritten queries
# If you pofile on the `base` workload type, no need to run `prepare-workload.sh` again
click-to-run/prepare-workload.sh [-tag <workload_type>]

click-to-run/profile-cost.sh -spes [-tag <workload_type>]
```

These scripts pick the optimized queries and profile them using Sql Server database.

Use `click-to-run/prepare-workload.sh` to prepare workload data in Sql Server database.

`click-to-run/estimate-cost.sh -spes` takes previously generated file `wtune_data/rewrite/result_spes/1_query.tsv` as input and
pick one rewritten query with the minimal cost by asking the database's cost model. The result will be stored
in `wtune_data/rewrite/result_spes/2_query.tsv` and used rules will be stored in
`wtune_data/rewrite/result_spes/2_trace.tsv`.

`click-to-run/profile-cost.sh -spes` evaluates the optimized queries. The output file is in `wtune_data/profile/result_spes/` by
default.

### View results
Finally, you can run:
```shell
click-to-run/view-all.sh -spes
```
to view rewriting and profiling results together.

Corresponding output files are in `wtune_data/viewall/result_spes/`.

## Run Examples

This section provides the instruction of run examples:

* template enumeration
* rule verification
* constraint enumeration

### Template Enumeration Example

```shell
click-to-run/run-template-example.sh
```

All templates of max size 4 (after pruning) will be printed, 3113 in total.

### Rule Verification

```shell
click-to-run/run-verify-example.sh [rule_index]
```

`<rule_index>` can be 1-35, corresponds to Table 7 in the paper.

For each rule, the following items will be printed:

* The rule itself.
* A query q0, on which the rule can be applied.
* A query q1, the rewrite result after applying the rule to q0.

For rule 1-31, which can be proved by WeTune built-in verifier, these additional items will be printed:

* A pair of U-Expression, translated from the rule.
* One or more Z3 snippets, the formula submitted to Z3.

> **Why there can be more than one snippet?**
>
> To relief the burden of Z3, When we are going to prove `(p0 /\ p1 ... /\ pn) /\ (q \/ r)` is UNSAT, we separately prove that `(p0 /\ p1 ... /\ pn) /\ q`
and `(p0 /\ p1 /\ ... /\ pn)` are both UNSAT. This is particularly the case when applying theorem 5.2.

### Constraint Enumeration

```shell
click-to-run/run-enum-example.sh [-dump] <rule_index>
```

`<rule_index>` can be 1-35, corresponds to Table 7 in the paper.

`-dump` specifies whether to dump all searched constraint sets to output.

WeTune will enumerate the constraints between the plan template of given rule, and search for the most-relaxed
constraint sets. Each of the examined constraint set and its verification result will be printed. The found rules and
metrics will be appended after enumeration completes.

P.S. If `-dump` is specified, for some pairs, the output floods for a few minutes, you may want to dump it to a file.

## Calcite Query Set

The Calcite queries can be found in `wtune_data/calcite/calcite_tests`.

### Rewrite Calcite Queries
We can use discovered rules to rewrite and profile Calcite queries as well,
simply by adding an optional parameter `-calcite` to related scripts in the workflow:

```shell
click-to-run/rewrite-queries.sh -calcite [-R <path/to/rules>]
```
The rewritten queries can be found in `wtune_data/calcite/result/`.


```shell
# Estimate the cost of rewritten queries and pick one with the minimal cost
# The prepared `base` workload is used to estimate the queries' cost
click-to-run/prepare-workload.sh -calcite -tag base
click-to-run/estimate-cost.sh -calcite

# Profile the performance of rewritten queries
# If you pofile on the `base` workload type, no need to run `prepare-workload.sh` again
click-to-run/prepare-workload.sh -calcite [-tag <workload_type>]

click-to-run/profile-cost.sh -calcite [-tag <workload_type>]
```
Use `click-to-run/prepare-workload.sh -calcite` to prepare workload data in Sql Server database.

`click-to-run/estimate-cost.sh -calcite` takes previously generated file `wtune_data/calcite/result/1_query.tsv` as input and
pick one rewritten query with the minimal cost by asking the database's cost model. The result will be stored
in `wtune_data/calcite/result/2_query.tsv` and used rules will be stored in
`wtune_data/calcite/result/2_trace.tsv`.

`click-to-run/profile-cost.sh -calcite` evaluates the optimized queries. The output file is in `wtune_data/profile_calcite/result/` by
default.

Finally, you can run:
```shell
click-to-run/view-all.sh -calcite
```
to view rewriting and profiling results together.

Corresponding output files are in `wtune_data/viewall_calcite/result/`.

### Test Verifier on Calcite Query Set

```shell
click-to-run/verify-calcite-query.sh
```

This script verify the equivalence of Calcite query pairs by directly transforming the entire query into rule. The
ordinal (line number) of 35 verifiable pairs will be printed, together with the rule.

```shell
click-to-run/verify-calcite-transformation.sh
```

This script verify the equivalence of Calcite query pairs by directly both of two queries and checking whether the
rewritten queries coincide, which effectively indicates WeTune can verify the transformation between the two query. The
ordinal (line number) of 73 verifiable pairs will be printed, together with the rewritten query.

