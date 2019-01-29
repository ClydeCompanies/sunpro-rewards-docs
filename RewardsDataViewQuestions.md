@darrenp, @joshm, and @de-wayned, I have some additional questions about the `Rewards_Data_View`:

* It (mostly) seems like I can use the `transaction_description` in combination with the `system_name` and/or `customer_master_id` as the unique identifier for an invoice so that I can track payments on it. Is this correct?

* There are some transactions that have a NULL `transaction_description`, yet have a `transaction_amount` from BisTrack. Do I need to assign my own unique identifier to these? For example, I could use the combination of `transaction_date` + `customer_master_id` + `transaction_original_amount` for that, but there is always a chance that even that combo will not be unique. I used the following queries to get this info:

```sql
-- Get a sample of this type of transaction record
SELECT TOP 1000 * FROM dbo.Rewards_Data_View WHERE
  (
    transaction_type_rewards_invoice != 0
    OR transaction_type_rewards_payment != 0
  )
  AND transaction_description IS NULL
  AND transaction_amount != 0;

-- Get a count of this type of transaction
SELECT COUNT(*) FROM dbo.Rewards_Data_View WHERE
  (
    transaction_type_rewards_invoice != 0
    OR transaction_type_rewards_payment != 0
  )
  AND transaction_description IS NULL
  AND transaction_amount != 0;

-- Get the system names (only BisTrack shows up) for this type of transaction
SELECT DISTINCT(system_name) FROM dbo.Rewards_Data_View WHERE
  (
    transaction_type_rewards_invoice != 0
    OR transaction_type_rewards_payment != 0
  )
  AND transaction_description IS NULL
  AND transaction_amount != 0;
```

* With retainage, is there a new transaction entry when it has been released? What will that look like? Or is this like the "retention" in the MySQL db, which (from what I can tell) represents a discount?


* When I run the following query, I find that there are transactions in the future (all on 2019-12-28, to be exact):

```sql
-- Are these recurring transactions or something like that? Should I ignore
-- them?
SELECT * FROM dbo.Rewards_Data_View WHERE
  (
    transaction_type_rewards_invoice = 1
    OR transaction_type_rewards_payment = 1
  )
  AND transaction_date >= GETDATE();
```
