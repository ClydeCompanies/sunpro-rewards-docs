# Sunroc Rewards Program Architecture

## Rewards Database Migration

An initial migration of data from the `rewards_program` MySQL database to the `sunroc_rewards` Postgres database. This migration will move over only those tables for which the new system will act as the system of record. This includes the following tables:

- `customers`
    - no need for `updatedYN10`, `du`, `unique_num`, `rewards_id`
- `divisions` - this could feasibly be moved to the code
- `prizes`
- `prizes_options` - rename to `prize_options`
- `prizes_redemptions` - rename to `prize_redemptions`
- `prizes_redemptions_batch` (not sure if this is needed)
- `property_development_points`
- `rewards` - change name to `customers_reward_groups`
- `rewards_favorites` (?) (seems to be an admin feature), change name to `reward_favorites`
- `rewards_groups` - change name to `reward_groups`
- `user_roles` (unless we use the code to determine permissions)
- `users`

## Website Architecture

The website will use Elixir/Phoenix for its backend. It will use an Elixir umbrella app, with the admin- and customer-facing portions using the Phoenix framework. The umbrella apps will be organized as follows:

```
sunroc_rewards/apps
├── data (Sunroc.Data) - Interfacing with the website's database.
├── notifications (Sunroc.Notifications) - Email notifications handler.
└── ui (Sunroc.UI) - The web application.
```

## Rewards Points Calculator Architecture

The current plan (as of Dec. 31, 2018) is to use an Elixir/Phoenix umbrella application to handle the processing of the invoice data. It will use the following pattern for the umbrella apps:

```
sunroc_invoice_processor/apps
├── calculator (Sunroc.Calculator) - Calculate points based on invoice data
│              and insert them into rewards database. Will have two repos for
│              connecting with each database.
└── dashboard (Sunroc.Dashboard) - Display statistics and status about data
              processing.
```

### Approach 1: Incremental Changes

* This is the currently selected option from Clyde Co.

`[PENDING DATA WAREHOUSE ACCESS FROM CLYDE CO]`

This approach will use the "event source" pattern. Each change is considered a transaction.

Changes are tracked, and points are calculated based on the changes, as logged in events. Events can be used as a log to show the reason that the points are what they are.

### Approach 2: Daily Recalculation

This approach is like the current setup in that it will erase all points and recalculate them anew each day. The system would be housed separately from the website's application and database to avoid performance issues.

The system will also separate logic from data; in other words, it won't use stored procedures to run the calculations, but will instead run the calculations via code.

Once calculations are complete, they will be copied over to the web application's database for consumption.
