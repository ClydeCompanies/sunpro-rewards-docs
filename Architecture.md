# Sunroc Rewards Program Architecture

## Website Architecture

The website should handle interfacing with the

* Ruby on Rails
* Elixir/Phoenix

## Rewards Points Calculator Architecture

### Approach 1: Incremental Changes

This approach will use the "event source" pattern. Each change is considered a transaction.

Changes are tracked, and points are calculated based on the changes, as logged in events. Events can be used as a log to show the reason that the points are what they are.

### Approach 2: Daily Recalculation

This approach is like the current setup in that it will erase all points and recalculate them anew each day. The system would be housed separately from the website's application and database to avoid performance issues.

The system will also separate logic from data; in other words, it won't use stored procedures to run the calculations, but will instead run the calculations via code.

Once calculations are complete, they will be copied over to the web application's database for consumption.
