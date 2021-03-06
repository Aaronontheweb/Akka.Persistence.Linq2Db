# Akka.Persistence.Linq2Db

A Cross-SQL-DB Engine Akka.Persistence plugin with broad database compatibility thanks to Linq2Db.

This is a port of the amazing akka-persistence-jdbc package from Scala, with a few improvements based on C# as well as our choice of data library.

Please read the documentation carefully. Some features may be specific to use case and have trade-offs (namely, compatibility modes)

## Status


- Implements the following for `Akka.Persistence.Query`:
    - IPersistenceIdsQuery
    - ICurrentPersistenceIdsQuery
    - IEventsByPersistenceIdQuery
    - ICurrentEventsByPersistenceIdQuery
    - IEventsByTagQuery
    - ICurrentEventsByTagQuery
    - IAllEventsQuery
    - ICurrentAllEventsQuery
- Snapshot Store Support

See something you want to add or improve? **Pull Requests are Welcome!**

Working:

- Persistence
- Recovery
- Snapshots


## Features/Architecture:

- Akka.Streams used aggressively for tune-able blocking overhead.
    - Up to `parallelism` writers write pushed messages
    - While writers are busy, messages are buffered up to `buffer-size` entries
    - Batches are flushed to Database at up to `batch-size`
        - For most DBs this will be done in a single built Multi-row-insert statement
        - PersistAll groups larger than `batch-size` will still be done as a single contiguous write

- Linq2Db usage for easier swapping of backend DBs.
    - `provider-name` is a `LinqToDb.ProviderName`
        - This handles DB Type mapping and Dialect-specific query building

- language-ext is used in place of Systems.Collections.Immutable where appropriate
    - Lower memory allocations, improve performance

- Recovery is also batched:
    - Up to `replay-batch-size` messages are fetched at a time
    - This is to both lower size of records fetched in single pass, as well as to prevent pulling too much data into memory at once.
    - If more messages are to be recovered, additional passes will be made.

- Attempts to stay in spirit and Structure of JDBC Port with a few differences:
    - Linq2Db isn't a Reactive Streams Compatible DB Provider (I don't know of any that are at this time for .NET)
        - This means Some of the Query architecture is different, to deal with required semantic changes (i.e. Connection scoping)
    - Both due to above and differences between Scala and C#, Some changes have been made for optimal performance (i.e. memory, GC)
        - Classes used in place of ValueTuples in certain areas
        - We don't have separate Query classes at this time. This can definitely be improved in future
        - A couple of places around `WriteMessagesAsync` have had their logic moved to facilitate performance (i.e. use of `await` instead of `ContinueWith`)
    - Backwards Compatibility mode is implemented, to interoperate with existing journals and snapsho stores.

## Currently Implemented:

- Journal
    - With `JournalSpec` and `JournalPerfSpec` passing for MS SQL Server, Microsoft.Data.SQLite, and Postgres
- Snapshot Store
    - With `SnapshotStoreSpec` passing for MS SQL Server, Microsoft.Data.SQLite, Postgres
- Configuration
    - Only Functional tests at this time.
    - Custom provider configurations are supported.

## Incomplete:

- Tests for Schema Usage
- Some Akka.NET specfic Journal Queries (those not mentioned above)
- Cleanup of Configuration classes/fallbacks.
    - Should still be usable in most common scenarios including multiple configuration instances: see `SqlServerCustomConfigSpec` for test and examples.

DB Compatibility:

- SQL Server: Tests Pass
- MS SQLite: Tests Pass
- System.Data.SQLite: Functional tests pass, perf tests partially fail.
- For whatever reason SDS doesn't cope well here.
- Postgres: Tests pass
- MySql: Not Tested Yet
- Firebird: Not Tested Yet


Compatibility with existing Providers is partially implemented via `table-compatibility-mode` flag. Please note there are not tests for compatibility with Sql.Common Query journals at this time:

- SQL Server: Basic Persist and Recovery tests pass for both Snapshot and Journal.
    - Still needs Delete tests
- SQLite: Persist and Recovery tests pass for both Snapshot and Journal.
- Postgres: Basic Persist and Recovery tests pass for both Snapshot and Journal.
    - Only Binary payloads are supported at this time.
    - Note that not all possible permutations of table names have been tested.
        - i.e. there may be concerns around casing
- MySql: Not Implemented yet

# Performance

Updated Performance numbers pending.

## Sql.Common Compatibility modes

- Delete Compatibility mode is available.
    - This mode will utilize a `journal_metadata` table containing the last sequence number
    - The main table delete is done the same way regardless of delete compatibility mode

###   *Delete Compatibility mode is expensive.*

- Normal Deletes involve first marking the deleted records as deleted, and then deleting them
    - Table compatibility mode adds an additional InsertOrUpdate and Delete
- **This all happens in a transaction**
    - In SQL Server this can cause issues because of pagelocks/etc.

## Configuration:

### Journal:

Please note that you -must- provide a Connection String (`connection-string`) and Provider name (`provider-name`).

- Refer to the Members of `LinqToDb.ProviderName` for included providers.
    - Note: For best performance, one should use the most specific provider name possible. i.e. `LinqToDB.ProviderName.SqlServer2012` instead of `LinqToDB.ProviderName.SqlServer`. Otherwise certain provider detections have to run more frequently which may impair performance slightly.

- `parallelism` controls the number of Akka.Streams Queues used to write to the DB.
    - Default in JVM is `8`. We use `3`
        - For SQL Server, Based on testing `3` is a fairly optimal number in .NET and thusly chosen as the default. You may wish to adjust up if you are dealing with a large number of actors.
            - Testing indicates that `2` will provide performance on par or better than both batching and non-batching journal.
        - For SQLite, you may want to just put `1` here, because SQLite allows at most a single writer at a time even in WAL mode.
            - Keep in mind there may be some latency/throughput trade-offs if your write-set gets large.
    - Note that unless `materializer-dispatcher` is changed, by default these run on the threadpool, not on dedicated threads. Setting this number too high may steal work from other actors.
        - It's worth noting that LinqToDb's Bulk Copy implementations are very efficient here, since under many DBs the batch can be done in a single async round-trip.
- `materializer-dispatcher` may be used to change the dispatcher that the Akka.Streams Queues use for scheduling.
    - You can define a different dispatcher here if worried about stealing from the thread-pool, for instance a Dedicated thread-pool dispatcher.
- `logical-delete` if `true` will only set the deleted flag for items, i.e. will not actually delete records from DB.
    - if `false` all records are set as deleted, and then all but the top record is deleted. This top record is used for sequence number tracking in case no other records exist in the table.
- `delete-compatibility-mode` specifies to perform deletes in a way that is compatible with Akka.Persistence.Sql.Common.
    - This will use a Journal_Metadata table (or otherwise defined )
    - Note that this setting is independent of `logical-delete`
- `use-clone-connection` is a bit of a hack. Currently Linq2Db has a performance penalty for custom mapping schemas. Cloning the connection is faster but may not work for all scenarios.
    - tl;dr - If a password or similar is in the connection string, leave `use-clone-connection` set to `false`.
    - If you don't have a password or similar, run some tests with it set to `true`. You'll see improved write and read performance.
- Batching options:
    - `batch-size` controls the maximum size of the batch used in the Akka.Streams Batch. A single batch is written to the DB in a transaction, with 1 or more round trips.
        - If more than `batch-size` is in a single `AtomicWrite`, That atomic write will still be atomic, just treated as it's own batch.
    - `db-round-trip-max-batch-size` tries to hint to Linq2Db multirow insert the maximum number of rows to send in a round-trip to the DB.
        - multiple round-trips will still be contained in a single transaction.
        - You will want to Keep this number higher than `batch-size`, if you are persisting lots of events with `PersistAll/(Async)`.
    - `prefer-parameters-on-multirow-insert` controls whether Linq2Db will try to use parameters instead of building raw strings for inserts.
        - Linq2Db is incredibly speed and memory efficent at building binary strings. In most cases, this will be faster than the cost of parsing/marshalling parameters by ADO and the DB.    
- For Table Configuration:
    - Note that Tables/Columns will be created with the casing provided, and selected in the same way (i.e. if using a DB with case sensitive columns, be careful!)

```hocon
akka.persistence {
  publish-plugin-commands = on
  journal {
    plugin = "akka.persistence.journal.linq2db"
    linq2db {
      class = "Akka.Persistence.Sql.Linq2Db.Journal.Linq2DbWriteJournal, Akka.Persistence.Sql.Linq2Db"
      plugin-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"
      connection-string = "" # Connection String is Required!

      # This dispatcher will be used for the Stream Materializers
      materializer-dispatcher = "akka.actor.default-dispatcher"
      
      # Provider name is required.
      # Refer to LinqToDb.ProviderName for values
      # Always use a specific version if possible
      # To avoid provider detection performance penalty
      # Don't worry if your DB is newer than what is listed;
      # Just pick the newest one (if yours is still newer)
      provider-name = "" 
      
      # If True, Deletes are done by updating Journal records
      # Rather than actual physical deletions
      logical-delete = false

      # If true, journal_metadata is created
      delete-compatibility-mode = true 

      # If "sqlite" or "sqlserver", and column names are compatible with
      # Akka.Persistence.Sql Default Column names.
      # You still -MUST- Set your appropriate table names!                       
      table-compatibility-mode = null

      #If more entries than this are pending, writes will be rejected.
      #This setting is higher than JDBC because smaller batch sizes
      #Work better in testing and we want to add more buffer to make up
      #For that penalty. 
      buffer-size = 5000

      #Batch size refers to the number of items included in a batch to DB
      #JDBC Default is/was 400 but testing against scenarios indicates
      #100 is better for overall latency. That said,
      #larger batches may be better if you have A fast/local DB.
      batch-size = 100

      #This batch size controls the maximum number of rows that will be sent
      #In a single round trip to the DB. This is different than the -actual- batch size,
      #And intentionally set larger than batch-size,
      #to help atomicwrites be faster
      #Note that Linq2Db may use a lower number per round-trip in some cases. 
      db-round-trip-max-batch-size = 1000

      #Linq2Db by default will use a built string for multi-row inserts
      #Somewhat counterintuitively, this is faster than using parameters in most cases,
      #But if you would prefer parameters, you can set this to true.
      prefer-parameters-on-multirow-insert = true

      # Denotes the number of messages retrieved
      # Per round-trip to DB on recovery.
      # This is to limit both size of dataset from DB (possibly lowering locking requirements)
      # As well as limit memory usage on journal retrieval in CLR
      replay-batch-size = 1000

      # Number of Concurrennt writers.
      # On larger servers with more cores you can increase this number
      # But in most cases 2-4 is a safe bet. 
      parallelism = 3
      
      #If a batch is larger than this number,
      #Plugin will utilize Linq2db's
      #Default bulk copy rather than row-by-row.
      #Currently this setting only really has an impact on
      #SQL Server and IBM Informix (If someone decides to test that out)
      #SQL Server testing indicates that under this number of rows, (or thereabouts,)
      #MultiRow is faster than Row-By-Row.
      max-row-by-row-size = 100
      
      #Only set to TRUE if unit tests pass with the connection string you intend to use!
      #This setting will go away once https://github.com/linq2db/linq2db/issues/2466 is resolved
      use-clone-connection = false
      
      tables.journal {

        #if delete-compatibility-mode is true, both tables are created
        #if delete-compatibility-mode is false, only journal table will be created.
        auto-init = true

        #
        table-name = "journal"
        metadata-table-name = "journal_metadata"
        
        #If you want to specify a schema for your tables, you can do so here.
        schema-name = null
        

        
        column-names {
          "ordering" = "ordering"
          "deleted" = "deleted"  
          "persistenceId" = "persistence_id"
          "sequenceNumber" = "sequence_number" 
          "created" = "created" 
          "tags" = "tags"
          "message" = "message"
          "identifier" = "identifier" 
          "manifest" = "manifest"
        }
        sqlserver-compat-column-names {
          "ordering" = "ordering"
          "deleted" = "isdeleted"
          "persistenceId" = "persistenceId"
          "sequenceNumber" = "sequenceNr"
          "created" = "Timestamp"
          "tags" = "tags"
          "message" = "payload"
          "identifier" = "serializerid"
          "manifest" = "manifest"
        }
        sqlite-compat-column-names {
          "ordering" = "ordering"
          "deleted" = "is_deleted"
          "persistenceId" = "persistence_Id"
          "sequenceNumber" = "sequence_nr"
          "created" = "Timestamp"
          "tags" = "tags"
          "message" = "payload"
          "identifier" = "serializer_id"
          "manifest" = "manifest"
        }
        metadata-column-names {
          "persistenceId" = "persistenceId"
          "sequenceNumber" = "sequenceNr"
        }
        sqlite-compat-metadata-column-names {
          "persistenceId" = "persistence_Id"
          "sequenceNumber" = "sequence_nr"
        }
      }
    }
  }
}

```

### Snapshot Store:

Please note that you -must- provide a Connection String and Provider name.

- Refer to the Members of `LinqToDb.ProviderName` for included providers.
    - Note: For best performance, one should use the most specific provider name possible. i.e. `LinqToDB.ProviderName.SqlServer2012` instead of `LinqToDB.ProviderName.SqlServer`. Otherwise certain provider detections have to run more frequently which may impair performance slightly.
```hocon
akka.persistence {
  snapshot-store
    {
      plugin = "akka.persistence.snapshot-store.linq2db"
      linq2db {
        class = "Akka.Persistence.Sql.Linq2Db.Snapshot.Linq2DbSnapshotStore"
        plugin-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"
        connection-string = ""
        provider-name = ""
        use-clone-connection = false
          
        #sqlserver, postgres, sqlite  
        table-compatibility-mode = false
        tables.snapshot
          {
            schema-name = null
            table-name = "snapshot"
            auto-init = true
            column-names {
              persistenceId = "persistence_id"
              sequenceNumber = "sequence_number"
              created = "created"
              snapshot = "snapshot"
              manifest = "manifest"
              serializerId = "serializer_id"
            }
          }
      }
    }
}
```

## Building this solution
To run the build script associated with this solution, execute the following:

**Windows**
```
c:\> build.cmd all
```

**Linux / OS X**
```
c:\> build.sh all
```

If you need any information on the supported commands, please execute the `build.[cmd|sh] help` command.

This build script is powered by [FAKE](https://fake.build/); please see their API documentation should you need to make any changes to the [`build.fsx`](build.fsx) file.

### Conventions
The attached build script will automatically do the following based on the conventions of the project names added to this project:

* Any project name ending with `.Tests` will automatically be treated as a [XUnit2](https://xunit.github.io/) project and will be included during the test stages of this build script;
* Any project name ending with `.Tests` will automatically be treated as a [NBench](https://github.com/petabridge/NBench) project and will be included during the test stages of this build script; and
* Any project meeting neither of these conventions will be treated as a NuGet packaging target and its `.nupkg` file will automatically be placed in the `bin\nuget` folder upon running the `build.[cmd|sh] all` command.

### DocFx for Documentation
This solution also supports [DocFx](http://dotnet.github.io/docfx/) for generating both API documentation and articles to describe the behavior, output, and usages of your project. 

All of the relevant articles you wish to write should be added to the `/docs/articles/` folder and any API documentation you might need will also appear there.

All of the documentation will be statically generated and the output will be placed in the `/docs/_site/` folder. 

#### Previewing Documentation
To preview the documentation for this project, execute the following command at the root of this folder:

```
C:\> serve-docs.cmd
```

This will use the built-in `docfx.console` binary that is installed as part of the NuGet restore process from executing any of the usual `build.cmd` or `build.sh` steps to preview the fully-rendered documentation. For best results, do this immediately after calling `build.cmd buildRelease`.

### Release Notes, Version Numbers, Etc
This project will automatically populate its release notes in all of its modules via the entries written inside [`RELEASE_NOTES.md`](RELEASE_NOTES.md) and will automatically update the versions of all assemblies and NuGet packages via the metadata included inside [`common.props`](src/common.props).

If you add any new projects to the solution created with this template, be sure to add the following line to each one of them in order to ensure that you can take advantage of `common.props` for standardization purposes:

```
<Import Project="..\common.props" />
```

### Code Signing via SignService
This project uses [SignService](https://github.com/onovotny/SignService) to code-sign NuGet packages prior to publication. The `build.cmd` and `build.sh` scripts will automatically download the `SignClient` needed to execute code signing locally on the build agent, but it's still your responsibility to set up the SignService server per the instructions at the linked repository.

Once you've gone through the ropes of setting up a code-signing server, you'll need to set a few configuration options in your project in order to use the `SignClient`:

* Add your Active Directory settings to [`appsettings.json`](appsettings.json) and
* Pass in your signature information to the `signingName`, `signingDescription`, and `signingUrl` values inside `build.fsx`.

Whenever you're ready to run code-signing on the NuGet packages published by `build.fsx`, execute the following command:

```
C:\> build.cmd nuget SignClientSecret={your secret} SignClientUser={your username}
```

This will invoke the `SignClient` and actually execute code signing against your `.nupkg` files prior to NuGet publication.

If one of these two values isn't provided, the code signing stage will skip itself and simply produce unsigned NuGet code packages.