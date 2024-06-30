Table generator CLI usage
=========================

```sh
dbgen -i template.sql -o out_dir -N 7500000 -R 300000 -r 100
```

Common options
--------------

* `-i «PATH»`, `--template «PATH»`

    The path to the template file. See [Template reference](Template.md) for details.

* `-o «DIR»`, `--out-dir «DIR»`

    The directory to store the generated files. If the directory does not exist, `dbgen` will try to
    create it.

* `-N «N»`, `--total-count «N»`

    Total number of rows to generate. Default is 1.

* `-R «N»`, `--rows-per-file «N»`

    Maximum number of rows per file generator thread. Default is 1.

* `-r «N»`, `--rows-count «N»`

    Maximum number of rows per INSERT statement. Default is 1.

The `--total-count` and `--rows-per-file` parameters also accept SI prefixes (e.g. `1.5K` = 1500)
and exponential form (e.g. `2.5e4` = 25000) to simplify some input.

More options
------------

* `-t «NAME»`, `--table-name «NAME»`

    Override the table name of generated data. Should be a qualified and quoted name like
    `'"database"."schema"."table"'`.

    This option cannot be used when the template has multiple tables.

* `--schema-name «NAME»`

    Replaces the schema name of the generated tables. Should be a qualified and quoted name like
    `'"database"."schema"'`.

* `--qualified`

    If specified, the generated INSERT statements will use the fully qualified table name (i.e.
    `INSERT INTO "db"."schema"."table" …`. Otherwise, only the table name will be included
    (i.e. `INSERT INTO "table" …`).

* `-s «SEED»`, `--seed «SEED»`

    Provide a 64-digit hex number to seed the random number generator, so that the output becomes
    reproducible. If not specified, the seed will be obtained from the system entropy.

    (Note: There is no guarantee that the same seed will produce the same output across major
    versions of `dbgen`.)

* `--rng «RNG»`

    Choose a random number generator. The default is `hc128` which should be the best in most
    situations. Supported alternatives are:

    | RNG name          | Algorithm             |
    |-------------------|-----------------------|
    | `chacha12`        | [ChaCha12][ChaCha20]  |
    | `chacha20`        | [ChaCha20]            |
    | `hc128`           | [HC-128]              |
    | `isaac`           | [ISAAC]               |
    | `isaac64`         | [ISAAC-64][ISAAC]     |
    | `xorshift`        | [Xorshift]            |
    | `pcg32`           | [PCG32]               |
    | `step`            | Step sequence         |

* `-j «N»`, `--jobs «N»`

    Use *N* threads to write the output in parallel. Default to the number of logical CPUs.

* `-q`, `--quiet`

    Disable progress bar output.

* `--escape-backslash`

    When enabled, backslash (`\`) is considered introducing a C-style escape sequence, and should
    itself be escaped as `\\`. In standard SQL, the backslash does not have any special meanings.
    This setting should match that of the target database, otherwise it could lead to invalid data
    or syntax error if a generated string contained a backslash.

    | SQL dialect | Should pass `--escape-backslash`                                |
    |-------------|-----------------------------------------------------------------|
    | MySQL       | Yes if [`NO_BACKSLASH_ESCAPES`] is off (default)                |
    | PostgreSQL  | No if [`standard_conforming_strings`] is on (default since 9.1) |
    | SQLite3     | No                                                              |
    | TransactSQL | No                                                              |

    Backslashes in template strings are always not special, regardless of this setting.

* `-k «N»`, `--files-count «N»`

    (Deprecated) Total number of file generator threads.

* `-n «N»`, `--inserts-count «N»`

    (Deprecated) Number of INSERT statements per file generator threads.

* `--last-file-inserts-count «N»`

    (Deprecated) In the last file generator thread, produce *N* INSERT statements instead of the
    value given by `--inserts-count`.

* `--last-insert-rows-count «N»`

    (Deprecated) In the last INSERT statement of the last file generator thread, produce *N* rows
    instead of the value given by `--rows-count`.

    These four options provide an alternative way to specify the total number of rows. But they are
    more difficult to manage, and are no longer recommended since v0.8.0.

* `--now «TIMESTAMP»`

    Override the timestamp reported by `current_timestamp`. Defaults to the time (in UTC) when
    `dbgen` was started. The timestamp must be written in the format `YYYY-mm-dd HH:MM:SS.fff`.

* `-e «TEMPLATE»`, `--template-string «TEMPLATE»`

    Pass the content of a template as an inline string via this command line argument.

* `-D «EXPR»`, `--initialize «EXPR»`

    Executes the global expression before generating files. This parameter can be specified multiple
    times. The expressions are executed before the global expressions in the template file. This
    allows user to parametrize the template, e.g. here we define a `@level` variable defaults to 1

    ```sql
    /*{{ @level := coalesce(@level, 1) }}*/
    create table foo (
        …
    ```

    and then we can use `-D` to override `@level`:

    ```sh
    ./dbgen -D '@level := 2' …
    ```

* `-f «FORMAT»`, `--format «FORMAT»`

    Output format of the data files. Could be one of:

    | Format            | Data output sample |
    |-------------------|--------------------|
    | sql               | <pre>INSERT INTO tbl (col1, col2) VALUES<br>(1, 'one'),<br>(3, 'three');</pre> |
    | csv               | <pre>"col1","col2"<br>1,"one"<br>3,"three"</pre> |
    | sql-insert-set    | <pre>INSERT INTO tbl SET<br>col1 = 1,<br>col2 = 'one';</pre> |

* `--format-true «STRING»`, `--format-false «STRING»`, `--format-null «STRING»`

    Change the string printed for TRUE, FALSE and NULL results.

    The default values are:

    | Format              | True | False | Null |
    |---------------------|------|-------|------|
    | sql, sql-insert-set | 1    | 0     | NULL |
    | csv                 | 1    | 0     | \\N  |

    Some database systems (e.g. PostgreSQL) distinguish between boolean and integer types. When
    targeting these systems, you may need to modify these keywords:

    ```sh
    ./dbgen -o bool_table -N 10 -R 10 \
        -e 'create table bool_table(a boolean {{ rand.bool(0.5) }});' \
        --format-true TRUE --format-false FALSE
    ```

    Regardless of these settings, when evaluating a boolean value as a string they always turn into
    `'0'` or `'1'` (e.g. `{{ true || '!' }}` always produces `'1!'`).

* `--headers`

    Include column names into the output as headers.

    In SQL format, this means insert statements include the complete column name list.

    ```sql
    INSERT INTO tbl (col1, col2, col3) VALUES
    (1, 2, 3),
    (4, 5, 6);
    ```

    In CSV format, this means files contain a header row of column names.

    ```csv
    "col1","col2","col3"
    1,2,3
    4,5,6
    ```

* `-c «ALG»`, `--compress «ALG»` / `--compress-level «LEVEL»`

    Compress the data output. Possible algorithms are:

    | Algorithm | Levels |
    |-----------|--------|
    | [gzip]    | 0–9    |
    | [xz]      | 0–9    |
    | [zstd]    | 1–21   |

    The compression level defaults to 6 if not specified.

    Since the data are randomly generated, the compression ratio is typically not very high (around
    70% of uncompressed input). We do not recommend using the algorithm "xz" here, nor using very
    high compression levels.

* `-z «SIZE»`, `--size «SIZE»`

    Target size (in bytes) of each data file. Default is unlimited.

    Explicit units are accepted (e.g. `8kB` = 8000, `0.5MiB` = 524288).

    When the target size is provided, each file generator thread may produce multiple files. After a
    complete INSERT statement (i.e. `--row-count` rows) is written, if the pre-compressed size of
    the file exceeds this `--size` parameter, dbgen will close it and start to write to a new file.

    The multiple files generated this way are written sequentially. You should still consider
    adjusting `--total-count`/`--rows-per-file` or `--files-count` to take advantage of parallelism.

    The size limits on a main table and its derived tables are considered independently. Thus, the
    number of files for them may not match.

    The file names are still ordered *lexicographically* but not necessarily numerically. Example
    directory is like:

    ```
    # suppose 10102 files are generated from file generator thread 0:
    tbl.0000.csv
    tbl.0001.csv
    ...
    tbl.0099.csv
    tbl.010000.csv
    tbl.010001.csv
    ...
    tbl.019999.csv
    tbl.02000000.csv
    tbl.02000001.csv

    # similar for file generator thread 1:
    tbl.1000.csv
    tbl.1001.csv
    ...
    tbl.1099.csv
    tbl.110000.csv
    tbl.110001.csv
    ...
    ```

    In lexicographic ordering, `tbl.1000.csv` should appear after `tbl.010000.csv`. But numerical or
    "natural" ordering will switch the order, and potentially affect subsequent import efficiency.

* `--components schema,table,data`

    What components to be generated:

    * `schema` (the `CREATE SCHEMA` SQL files)
    * `table` (the `CREATE TABLE` SQL files)
    * `data` (the output files)

[ChaCha20]: https://cr.yp.to/chacha.html
[HC-128]: https://www.ntu.edu.sg/home/wuhj/research/hc/index.html
[ISAAC]: http://www.burtleburtle.net/bob/rand/isaacafa.html
[Xorshift]: https://en.wikipedia.org/wiki/Xorshift
[PCG32]: http://www.pcg-random.org/
[gzip]: https://en.wikipedia.org/wiki/Gzip
[xz]: https://en.wikipedia.org/wiki/Xz
[zstd]: https://facebook.github.io/zstd/

[`NO_BACKSLASH_ESCAPES`]: https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_no_backslash_escapes
[`standard_conforming_strings`]: https://www.postgresql.org/docs/current/static/runtime-config-compatible.html#GUC-STANDARD-CONFORMING-STRINGS