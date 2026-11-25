## Start with one call

```sql
BEGIN
  dbms_vector.rebuild_index(
    idx_name => 'DEMO_HNSW_IDX',
    options  => JSON('{"idx_parameters":{"neighbors":12}}'));
END;
/
```

Suppose the stored HNSW index has `NEIGHBORS 8` and `EFCONSTRUCTION 64`. The requested change is `NEIGHBORS 12`. The ordinary compact rebuild reads the current index definition, replaces the neighbor count, and carries forward the stored construction setting. This is why the code needs both the supplied JSON and the existing definition.

## Public names and private work

A PL/SQL **procedure** performs an operation. A **function** returns a value. An **overload** gives callers multiple argument lists for the same public name. PL/SQL chooses the declaration that matches the call. Here, the named argument `options =>` selects the compact overload. There are two new declarations, one for create and one for rebuild. Together they cover four combinations with HNSW and IVF.

The package specification declares the public procedures and the five JSON constants. The package body defines the procedures and the private helpers they call. In the first implementation, `impl` stood for *implementation*. The refactor gives the shared routines verb names that describe their work.

| Name in the body | How to read it |
| --- | --- |
| `CREATE_INDEX` | Two separate public procedures. Each prepares clauses from its own arguments. Both call `execute_create_index`. |
| `execute_create_index` | A private package procedure that assembles and runs one `CREATE VECTOR INDEX` statement. |
| `REBUILD_INDEX` | Two separate public procedures. The flat overload resolves the owner and calls the existing `rebuild_index_for_owner` entry point. The compact overload reads JSON and resolves the owner. |
| `execute_rebuild_index` | The shared procedure that checks privileges, chooses the rebuild mode, and performs the operation. |
| `apply_flat_rebuild_options`, `apply_compact_rebuild_options` | Local procedures inside `execute_rebuild_index`. Each edits the same local `index_ddl` variable according to its caller’s rules. |

A line beginning with `PROCEDURE execute_create_index(...)` **defines** the helper. The line `execute_create_index(idx_name, table_name, idx_vector_col, clauses);` **calls** it. The two public create procedures and the private executor are siblings in the package body. Each ends with its own `END procedure_name;`.

Some shorter names predate this change. In this package, `idx_` marks an index argument, `l_` marks a local variable, and `uq_` identifies an unquoted name used to look up metadata. The public template constants start with `C_`. DDL means *data definition language*, the SQL statements that create or alter database objects. Thus `index_ddl` holds SQL text, while `supplied` holds the validated JSON options. A suffix such as `_sql` identifies text suitable for a SQL statement.

[Public specification](../admin/dbmsvector.sql) · [Package body](../src/server/dict/plsicds/prvtvector.sql)

## What changed and why

| Change | Reason |
| --- | --- |
| Each create overload prepares a private `create_index_clauses` record. | The flat API receives PL/SQL defaults such as accuracy 90. Compact create lets SQL choose the default when a field is missing. You can now read either set of rules from start to finish. |
| One create executor quotes the target names and emits the clauses in SQL order. | The two APIs need the same identifier handling, clause order, tracing of the executed statement, and execution call. A fix to that SQL assembly belongs in one place. |
| `read_index_options` walks the corresponding create or rebuild templates. | The HNSW and IVF templates give the allowed keys at each depth. Checking the key first catches a typo even if its value is null. Dropping JSON nulls and empty child objects lets later code use one test for a supplied option. |
| `flat_index_parameter_list` serves flat create and ordinary flat rebuild. | The HNSW formatting was duplicated. The helper keeps their existing difference for `efConstruction` and reuses the IVF parameter builder. The shared append routine now has the general name `append_index_parameter`. |
| `compact_parameter_list` formats supplied fields and can merge them with a stored list. | On a same-type rebuild it carries forward stored fields the caller omitted. On a type switch it starts a list for the requested organization. SQL spells the JSON key `partitions` as `NEIGHBOR PARTITIONS`. |
| `replication_method_sql` translates both APIs’ method names. | Both APIs can map `PARTITION` directly to `DISTRIBUTE BY PARTITION`. The compact reader can pass the method directly, without constructing a second JSON object for the flat API to parse. |
| Ordinary rebuild chooses one editor immediately after saving the original DDL. | The flat API accepts a complete replacement parameter list. The compact API accepts partial updates. Keeping these edits together gives each API one readable path while sharing privilege checks and recovery. |
| Compact rebuild locates SQL clauses outside quoted text. | An index named `"ONLINE"` must keep its name when a caller changes the `ONLINE` clause. The editor can also insert a clause that is absent from the stored DDL. |
| Explicit rebuild modes continue through the existing HNSW or IVF operations. | A global IVF `FULL` rebuild uses `ALTER INDEX`. A compact request containing only `"type":"IVF"` has no parameter to change, so the generated statement omits `PARAMETERS`. |

### Create and rebuild defaults

The flat and compact overloads have different input contracts. On compact **create**, a missing field and a JSON null both omit the corresponding SQL clause. Passing no `options` argument works. On ordinary compact **rebuild**, both forms retain that setting from the stored definition. The options argument is required so the existing one-argument `REBUILD_INDEX('IDX')` call continues to select the flat overload. Use `C_RETAIN_INDEX_CONFIG` for the compact call that requests no changes.

The four public organization-specific constants are copyable templates. The two rebuild templates explicitly set an organization and a parameter type. Use the empty retain constant when the existing organization should determine the result. Preservation on the ordinary path comes from the DDL returned by `DBMS_METADATA.GET_DDL`. Build-time inputs such as the offload URL and credential are ephemeral.

### JSON conversion and SQL formatting

| Field | Accepted input |
| --- | --- |
| `idx_accuracy`, `idx_parallel_creation`, `idx_compression_ratio` | A number or numeric text convertible by PL/SQL, matching the flat numeric arguments. |
| `idx_parameters.efConstruction` | Create accepts a number or convertible numeric text. Rebuild requires a JSON number. |
| Other numeric index parameters, such as `neighbors` | A JSON number. |
| `idx_online_build` | A JSON boolean such as `true` or `false`. |

The reader checks keys and JSON types before assembling SQL. It uses the union of keys from both organizations so the SQL engine can check index-specific combinations. Existing mode-specific restrictions and privilege checks still apply. SQL identifiers pass through the relevant `DBMS_ASSERT` routines. A compact offload URL is quoted as a string literal with embedded single quotes doubled. The compact numeric formatter emits a period for the SQL decimal separator.

For replication, one object contains a mode and its parameters. For example, `{"idx_replication":{"mode":"DUPLICATE","parameters":{"method":"ALL","service_name":"VEC_SERVICE"}}}` produces `DUPLICATE ALL FOR SERVICE "VEC_SERVICE"`. A service-only compact rebuild can take the mode and method from the stored clause. Creating a replication clause from scratch needs a mode.

## Appendix A — Follow the code

**Compact create.** After validation, only supplied keys remain. This excerpt from the public compact overload explains exactly when the accuracy clause enters the record.

```plsql
IF supplied.has('idx_accuracy') THEN
  clauses.accuracy := 'TARGET ACCURACY ' ||
                      index_number_text(supplied.get_number('idx_accuracy'));
END IF;
```

Both public create procedures finish with the same call. The executor uses the presence of accuracy or distance to emit `WITH`, so a compact call can supply distance alone.

```plsql
execute_create_index(idx_name, table_name, idx_vector_col, clauses);
```

**Ordinary rebuild.** The executor reads the index’s DDL only after the privilege checks. A flat call passes a null `compact_options` object. A compact call always passes an object, even when it is empty. This excerpt is the single point where the executor chooses how to edit that DDL.

```plsql
index_ddl_orig := index_ddl;
IF compact_options IS NULL THEN
  apply_flat_rebuild_options;
ELSE
  apply_compact_rebuild_options;
END IF;
```

After editing, the executor drops and recreates the index. If creation fails after the drop, the error handler attempts to recreate the original definition and re-raises the error. This path performs DDL and can commit. Explicit `FULL` and `INCREMENTAL` modes take the index-specific paths described above.

## Appendix B — Exercise all four combinations

These examples are for a disposable, vector-enabled development schema with the updated package installed and sufficient vector memory for HNSW. They create two tables and two indexes. Run the metadata query before and after the rebuilds so you can see which settings changed. The setup assumes the demo object names are available.

```sql
CREATE TABLE compact_demo_h (embedding VECTOR(3, FLOAT32));
CREATE TABLE compact_demo_i (embedding VECTOR(3, FLOAT32));

INSERT INTO compact_demo_h
  SELECT TO_VECTOR('[' || TO_CHAR(LEVEL, 'FM9990') || ',1,0]')
  FROM dual CONNECT BY LEVEL <= 256;
INSERT INTO compact_demo_i SELECT embedding FROM compact_demo_h;
COMMIT;

BEGIN
  dbms_vector.create_index(
    idx_name => 'DEMO_HNSW_IDX', table_name => 'COMPACT_DEMO_H',
    idx_vector_col => 'EMBEDDING', idx_organization => 'INMEMORY NEIGHBOR GRAPH',
    options => JSON('{"idx_parameters":{"type":"HNSW","neighbors":8,"efConstruction":64}}'));

  dbms_vector.create_index(
    idx_name => 'DEMO_IVF_IDX', table_name => 'COMPACT_DEMO_I',
    idx_vector_col => 'EMBEDDING', idx_organization => 'NEIGHBOR PARTITIONS',
    options => JSON('{"idx_parameters":{"type":"IVF","partitions":2}}'));
END;
/

SET LONG 100000
SELECT dbms_metadata.get_ddl('VECTOR_INDEX', index_name)
FROM user_indexes
WHERE index_name IN ('DEMO_HNSW_IDX', 'DEMO_IVF_IDX')
ORDER BY index_name;

BEGIN
  dbms_vector.rebuild_index(
    idx_name => 'DEMO_HNSW_IDX',
    options => JSON('{"idx_parameters":{"neighbors":12,"efConstruction":null}}'));

  dbms_vector.rebuild_index(
    idx_name => 'DEMO_IVF_IDX',
    options => JSON('{"idx_parallel_creation":2}'));
END;
/

SELECT dbms_metadata.get_ddl('VECTOR_INDEX', index_name)
FROM user_indexes
WHERE index_name IN ('DEMO_HNSW_IDX', 'DEMO_IVF_IDX')
ORDER BY index_name;
```

For HNSW, look for `NEIGHBORS 12` and `EFCONSTRUCTION 64`. For IVF, look for `PARALLEL 2` and compare the stored partition setting with its value before the rebuild. To check the retain constant, call `dbms_vector.rebuild_index('DEMO_IVF_IDX', options => dbms_vector.C_RETAIN_INDEX_CONFIG)` and compare the metadata again.

The existing sanity suite provides examples of flat calls and their expected results. For regression coverage, also check a four-argument compact create, a one-argument flat rebuild, an unknown nested key with a null value, numeric text on both create and rebuild, and a quoted identifier containing a clause keyword. The local instance returned `ORA-01034` during this refactor. The four examples above still need database execution.

Offline verification covered specification/body signature matching, all four templates, and five create-statement assembly cases. Oracle's PL/SQL wrapper processed both source files. Package compilation against the database catalog and live rebuild behavior still need verification in the development database.

[Existing sanity SQL](../../tkmain_1/tkvc/sql/tkdbcvdbpck_sanity.sql) · [Expected output](../../tkmain_1/tkvc/log/tkdbcvdbpck_sanity.log)
