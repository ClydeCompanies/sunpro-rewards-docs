# Rewards Calculation System

The process of calculating points is a complex system of applications and data sources that orchestrates the calculation of rewards points and penalties. There are three main data sources in this system:

1. The `Rewards_Data_View` SQL Server data view
    - For the purposes of the rewards program, this is the source of truth for all transactions.
2. The `sunpro_transactions` Postgres database
    - This is an import of transactions for active rewards groups. Having the transactions in this database allows us to have an easily-queriable backlog of transactions to import into the main rewards system over time, and to mark transactions as processed once they've been inserted into the rewards system. This also allows us to use more up-to-date versions of Elixir packages when doing more complex work on the transactions, since the packages for querying an SQL Server database are outdated.
3. The `sunpro_rewards` Postgres database
    - This is the main application's database, and consists of transactions, invoices (which are groups of transactions), points, redemptions, and other rewards group- and rewards-related entities.

And there are three main applications:

1. `sunpro_transactions`
    - This is an application that runs on the application server. It has a worker pool that runs through jobs asynchronously. Its responsibility is mostly to move data from the data warehouse to the `sunpro_transactions` Postgres database.
1. `sunpro_transactions_processor`
    - This is an application that runs on the application server. It has a worker pool that runs through jobs asynchronously. Its main responsibilities are to import transactions into the rewards system and calculate their points.
1. `sunpro_rewards`
    - This application is a web service backed by the `sunpro_rewards` Postgres database. Its job is to provide a user interface for clients and admins to view and manage points.

## Flow of Data

The transactions flow from the `Rewards_Data_View` data view in the data warehouse to the `sunpro_transactions` Postgres database, and then from there are processed, converted, and inserted into the `sunpro_rewards` database.

1. `Rewards_Data_View` -> `sunpro_transactions`
    - The transaction rows are queried by `updated_date`, `customer_master_id`, and `transaction_date` (not always all at the same time), and copied into the `sunpro_transactions` database almost verbatim. The only differences are that the `sunpro_transactions` database is a Postgres database and that it includes a `processed_at` column for each transaction. This is to keep track of whether or not the transaction has been copied over to the `sunpro_rewards` database.
2. `sunpro_transactions` -> `sunpro_rewards`
    - The process of importing transaction rows from the `sunpro_transactions` database to the `sunpro_rewards` database involves simplifying the column names, finding or inserting an invoice to attach the transaction to, and attaching the transaction to the other relevant entities (companies, product lines, systems, and customers). Importing a transaction also changes the amount and balance of the associated invoice. The points for the transaction will be calculated at a later point.

## Import Logic

In order to minimize the number of calculations that happen each day, the transactions are limited according to their `updated_date`. The logic works like this:

1. The `sunpro_transactions` application imports transactions from the `Rewards_Data_View` to the `sunpro_transactions` database:
    - A `TransactionImportWorker` is scheduled every thirty minutes on the 7th and 37th minute of the hour.
    - This worker doesn't take any arguments, and delegates transaction imports to other workers.
    - It schedules another `TransactionImportWorker` for each active rewards group.
    - It also enqueues a worker for each day since the program's inception for each newly added rewards group.
    - The active rewards groups use the `last_updated_date` to run a query to retrieve any new transactions.
    - Transactions that have already been imported, but are being imported again (probably because they have been updated and/or paid off), are updated in the `sunpro_transactions` database, and the `processed_at` value is set to `NULL`.
2. Unprocessed transactions are imported into the rewards system:
    - While transactions are imported into the `sunpro_transactions` database, workers in the `sunpro_transactions_processor` application retrieve unprocessed transactions and upsert them into the `sunpro_rewards` database, associating them as appropriate to other entities.
    - The worker initially divides up the work between other workers by master ID.
    - Each worker for a particular master ID works in batches of 100, and re-enqueues itself if it processed any transactions.
    - Every 15 seconds, the system checks if there are transaction import workers enqueued, and if there aren't any, it enqueues a new one with no arguments.
3. An `InvoiceWorker` processes invoices and calculates points and penalties.
    - It is scheduled to run every 5 minutes.
    - With no arguments, it schedules `InvoiceWorker` instances for all rewards groups that have unprocessed invoices, overdue invoices, or paid invoices.
    - For each rewards group, the worker works in batches of 10 invoices per run.

## Syncing Logic

1. The `sunpro_transactions` application syncs customer and rewards group information from the `Customer_Dimension` table (view?):
    - The `RewardsGroupImportWorker` runs once per hour on the 0th minute.
    - The `CustomerImportWorker` runs once per hour on the 30th minute.
2. The `sunpro_transactions_processor` application syncs rewards group information from the `sunpro_rewards` database into the `sunpro_transactions` database.
    - The `RewardsGroupSyncWorker` runs every 15 minutes on the 5th, 20th, 35th, and 50th minutes. It syncs the enrolled at date and whether or not the rewards group is active.

## Worker Configuration

### Worker Concurrency

Worker concurrency is configured at compile time with the `TRANSACTIONS_WORKER_CONCURRENCY` environment variable. It is set to 5 by default, which means that 5 worker instances are always available in the worker pool. There is one worker pool in the `sunpro_transactions` application, and one worker pool in the `sunpro_transactions_processor` application.

### Rules of Job Execution

Jobs are database records that help determine which workers should be run when and with which arguments.

Workers are executed based on the `next_run_at` datetime recorded on its job record. The rows with the oldest `next_run_at` datetimes are run first, as long as the `next_run_at` is not in the future. When there are many jobs scheduled for the same `next_run_at` datetime, the ordering of their execution is non-deterministic.

If there is no work currently scheduled in the `jobs` table, the workers all sleep for 15 seconds. After the 15 seconds, they each check if there is work to be done, and if there is not, they sleep again.

Jobs are locked in a transaction when a worker retrieves it so that other workers won't pick it up. This prevents multiple workers from working on the same job at once.

### Scheduled Workers

Some recurring workers are scheduled for particular runtimes. Every fifteen seconds, the system checks if those workers have been enqueued, and if not, enqueues them. If there are many other jobs that have to run before that worker (based on the `next_run_at` datetime), that recurring worker may be delayed until the other jobs have been run.

## Timing

Because the system uses a jobs and workers system to import and process transactions, the exact amount of time between a transaction being updated or inserted into `Rewards_Data_View` and being imported and processed in the Rewards application is non-deterministic. In a typical scenario where there are no new rewards groups added and a reasonable number of newly updated transactions, it should take between 1 and 35 minutes to import a transaction to the `sunpro_transactions` database. This is because the transaction import worker runs every 30 minutes, and it could take about 5 minutes to get through all the transactions that need to be imported. In the worst case scenario (a trunc and reload), this could be many hours.

Once the transaction is imported into the `sunpro_transactions` database, it should be almost immediately imported into the `sunpro_rewards` database. This is because the `TransactionImportWorker` in the `sunpro_transactions_processor` app tries to schedule itself every 15 seconds. In a typical scenario, however, it could take 5-10 minutes for a particular transaction to be imported, depending on when the transaction importer ends up running for that particular rewards group.

Since the `InvoiceWorker` runs every 5 minutes, and there may be many invoices to process, actual points calculation can take between 1 and 30 minutes to take place.

Putting the above all together, it can take between 2 minutes and 1 hour and 15 minutes in a typical scenario for a particular transaction to be imported and its points processed.

## Calculation Logic

[TODO]
