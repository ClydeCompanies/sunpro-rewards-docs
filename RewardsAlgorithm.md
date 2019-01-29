# Sunroc Rewards Program Algorithm

## New System

### Points Calculation Algorithm

The following algorithm is centered around an event source paradigm. Transactions are pulled in from the `Rewards_Data_View` MSSQL view each [pre-specified period], and these transactions are possible sources for accruing or penalizing points. The point accruals and penalties will be recorded as events within the rewards database, and will be tied to a particular invoice. The invoice record will have information about its balance, due date, and amount. The algorithm will work like this:

1. From `Rewards_Data_View`, retrieve all transactions for the current date (assuming current date is >= 2011-01-01), ordered by invoices first, followed by payments.
2. Retrieve next transaction in list, and run the following:
    a. Generate a hash for the transaction using all fields in the row:
        `transaction_date`, `transaction_type_rewards_invoice`, `transaction_type_rewards_payment`, `transaction_line_number`, `transaction_contract`, `transaction_description`, `transaction_amount`, `transaction_original_amount`, `transaction_retainage`, `customer_id`, `customer_name`, `company_abbreviation`, `name`, `system_name`, `product_line_name`, `customer_master_id`
    b. If transaction hash exists in database already, go to 2.
    c. If transaction date < 2011-10-01 AND rewards group enrollment date >= 2011-10-01, go to 2.
    d. If transaction date < rewards group enrollment date AND rewards group enrollment date >= 2011-10-01, go to 2.
    e. Insert transaction record into `sunroc_rewards` database with unique hash, auto-incremented integer ID, and false (0) for processed.
    f. Go to 2.
3. Retrieve all unprocessed transactions from `sunroc_rewards` db.
4. Retrieve next unprocessed transaction from list retrieved in 3, and do the following:
    a. Try to retrieve invoice from `sunroc_rewards` database using business key.
    b. If no invoice, create new invoice record, using the business key as a unique ID.
    c. If transaction is invoice, add `transaction_amount` to balance of invoice.
    d. If transaction is payment, subtract `transaction_amount` from balance.
5. Retrieve all penalty- or accrual-eligible invoices from the `sunroc_rewards` DB. An invoice is eligible if:
    a. It is paid in full AND points have not been applied AND paid on date is before the last day of the month of the due date for that invoice.
    b. OR it is not paid in full AND
        - Its due date is 31 days or more ago AND its first penalty has not been applied
        - Its due date is 61 days or more ago AND its second penalty has not been applied
        - Its due date is 91 days or more ago AND its third penalty has not been applied
        - Its due date is 121 days or more ago AND its fourth penalty has not been applied
6. Retrieve next item in list of invoices obtained from 5.
    a. If invoice is now paid in full, but wasn't previously, and the current date is before or on the last day of the month of the due date, create points record and save to `sunroc_rewards` DB.
    b. If < 50% of invoice has been paid AND the due date month has past AND (all of the below that apply):
        - It is >= 31 days past the due date AND the first penalty has not been applied, then apply it
        - It is >= 61 days past the due date AND the second penalty has not been applied, then apply it
        - It is >= 91 days past the due date AND the third penalty has not been applied, then apply it
        - It is >= 121 days past the due date AND the fourth penalty has not been applied, then apply it

An initial migration will run the above algorithm for each date starting from 2011-01-01 and going until today's date. In this initial migration, point redemptions and manual adjustments will need to happen for each day so that points are calculated correctly.

### Database Schemata

The following are the database schemata for the rewards and invoices data, used for calculating, managing, and viewing reward points.

#### Invoices and Transactions

DB: sunroc_transactions

* `id` INT AUTOINCREMENT
* `hash` VARCHAR(64) - A SHA256-generated hash based on the fields.
* `business_key` VARCHAR(50)
* `transaction_date` DATETIME
* `transaction_type_rewards_invoice` BIT
* `transaction_type_rewards_payment` BIT
* `transaction_line_number` INT
* `transaction_contract` VARCHAR(50)
* `transaction_description` TEXT
* `transaction_amount` DECIMAL(15,2)
* `transaction_original_amount` DECIMAL(15,2)
* `transaction_retainage` DECIMAL(15,2)
* `customer_id` VARCHAR(50)
* `customer_name` TEXT
* `company_abbreviation` VARCHAR(3)
* `name` VARCHAR(60)
* `system_name` VARCHAR(50)
* `product_line_name` VARCHAR(50)
* `customer_master_id` TEXT
* `processed` BOOLEAN

#### Rewards

DB: sunroc_rewards

* Invoices:
    * Table: `invoices`
        * `id` UNSIGNED INT AUTOINCREMENT (PRIMARY KEY)
        * `business_key` VARCHAR(50)
        * `customer_id` VARCHAR(255)
        * `customer_master_id` VARCHAR(255)
        * `company_id` UNSIGNED IN (FOREIGN KEY)
        * `system_id` UNSIGNED INT (FOREIGN KEY)
        * `product_line_id` UNSIGNED INT (FOREIGN KEY)
        * `amount` DECIMAL(15,2)
        * `balance` DECIMAL(15,2)
        * `retainage` DECIMAL(15,2)
        * `points_granted` BOOLEAN
        * `penalty1_deducted` BOOLEAN
        * `penalty2_deducted` BOOLEAN
        * `penalty3_deducted` BOOLEAN
        * `penalty4_deducted` BOOLEAN
        * timestamps
    * Table: transactions
        * `id` UNSIGNED INT AUTOINCREMENT (PRIMARY KEY)
        * `hash` VARCHAR(64) - A SHA256-generated hash based on the fields
        * `invoice_id` UNSIGNED INT (FOREIGN KEY)
        * `business_key` VARCHAR(50)
        * `transaction_date` DATETIME
        * `transaction_type_rewards_invoice` BOOLEAN
        * `transaction_type_rewards_payment` BOOLEAN
        * `transaction_line_number` INT
        * `transaction_contract` VARCHAR(255) (INT?)
        * `transaction_description` VARCHAR(255)
        * `transaction_amount` DECIMAL(15,2)
        * `transaction_original_amount` DECIMAL(15,2)
        * `transaction_retainage` DECIMAL(15,2)
        * `customer_id` VARCHAR(255)
        * `company_id` UNSIGNED IN (FOREIGN KEY)
        * `system_id` UNSIGNED INT (FOREIGN KEY)
        * `product_line_id` UNSIGNED INT (FOREIGN KEY)
        * timestamps
* User Management and Permissions:
    * Table: `users`
    * ¿Table: `roles`?
    * ¿Table: `settings`?
* Customer Records
    * Table: `customers`
        * `id` VARCHAR(23)
        * `code` VARCHAR(255)
        * `name` TEXT
        * `status` VARCHAR(10)
        * `division_id`
        * `reward_group_id`
        * timestamps
    * Table: `reward_groups`
        * `id` UNSIGNED INT AUTOINCREMENT (PRIMARY KEY)
        * `name` TEXT
        * `status` VARCHAR(7)
        * `enrolled_at` DATETIME (from `created_on` in `rewards_program` db)
        * timestamps
* Clyde Co Entities:
    * Table: `companies`
        * `id` UNSIGNED INT AUTOINCREMENT (PRIMARY KEY)
        * `code` VARCHAR(16)
        * `name` VARCHAR(255)
        * `sort_order` INT
        * timestamps
    * Table: `divisions`
        * `id` INT AUTOINCREMENT (PRIMARY KEY)
        * `code` VARCHAR(5)
        * `name` TEXT
        * `company_id` INT (FOREIGN KEY)
        * `sort_order` INT
        * timestamps
    * Table: `systems`
        * `id` UNSIGNED INT AUTOINCREMENT (PRIMARY KEY)
        * `name` VARCHAR(50)
        * timestamps
    * Table: `product_lines`
        * `id` UNSIGNED INT AUTOINCREMENT (PRIMARY KEY)
        * `name` VARCHAR(255)
        * timestamps
* Points
    * Table: `point_transactions`
        * `id` UNSIGNED INT AUTOINCREMENT (PRIMARY KEY)
        * `type` VARCHAR(50) enum of (reward, penalty, adjustment, redemption)
        * `model_id` - Must be set for adjustment and redemption transactions. Must NOT be set for reward and penalty transactions.
        * `model_type` - Must be set for adjustment and redemption transactions. Must NOT be set for reward and penalty transactions.
        * `division_id`
        * `amount` DECIMAL(15,2)
        * timestamps
    * Table: `points`
        * `id`
        * `reward_group_id`
        * `company_id`
    * Table: `prize_redemptions`
    * Table: `point_adjustments`
    * Table: `property_development_points`
* Prizes
    * Table `prizes`
    * Table `prize_options`

### Database Migration

The `rewards_program` database will need to be migrated to the `sunroc_rewards` db before launch.

* `customers`
    * `customer_id` -> `id`
    * `customer_cd` -> `code`
    * `customer_name` -> `name`
    * `account_job_flag` -> REMOVE
    * `division_cd` -> `division_id`
    * `master_acct_id` -> `master_id`
    * `status_cd` -> `status`
    * `rewards_id` -> `reward_group_id`
    * `du` -> REMOVE
    * `updatedYN10` -> REMOVE
    * `unique_num` -> REMOVE
* `divisions`
    * CREATE `id`
    * `division_cd` -> `code`
    * `division_name` -> `name`
    * `sort_order` -> FLOAT TO INT
* `points_adjustments` -> `point_adjustments`
    * `pa_id` -> `id`
    * `rewards_id` -> `reward_group_id`
    * `division_cd` -> `division_id` (convert)
    * `description`
    * `points`
    * `created_by` -> `created_by_id`
    * `created_on` -> timestamps
* `prizes`
    * `prize_id` -> `id`
    * `prize_title` -> `title`
    * `points_value_input`
    * `sort_order` -> INT
* `prizes_options` -> `prize_options`
    * `po_id` -> `id`
    * `prize_id`
    * `option_title` -> `title`
    * `option_points_cost` -> `point_cost`
    * `option_dollar_value` -> `dollar_value`
    * `option_expire_date` -> `expires_at`
    * `option_sort_order` -> `sort_order` (INT)
    * `option_info_required` -> `info_required`
    * `option_international_info_required` -> `international_info_required`
* `prizes_redemptions` -> `prize_redemptions`
    * `pr_id` -> `id`
    * `u_id` -> `user_id`
    * `rewards_id` -> `reward_group_id`
    * `prize_id`
    * `po_id` -> `prize_option_id`
    * `points`
    * `cost_sbm`
    * `cost_src`
    * `cost_grp`
    * `cost_wwc`
    * `description`
    * `name_first`
    * `name_last`
    * `dob` -> `birthdate`
    * `phone`
    * `email`
    * `international_name`
    * `passport_number`
    * `password_issue_date`
    * `password_expiration_date`
    * `address`
    * `city`
    * `state`
    * `zip`
    * `gender`
    * `qty`
    * `created_on` -> timestamps
* `property_development_points`
    * `pdp_id` -> `id`
    * `rewards_id` -> `reward_group_id`
    * `division_cd` -> convert to `division_id`
    * `points`
    * `project_name`
    * `lot`
    * `created_by` -> `created_by_id`
    * `created_on` -> timestamps
* `rewards_favorites` -> `reward_group_favorites`
    * `u_id` -> `user_id`
    * `rewards_id` -> `reward_group_id`
* `rewards_groups` -> `reward_groups`
    * `rewards_id` -> `id`
    * `rewards_name` -> `name`
    * `created_on` -> `enrolled_at` + timestamps
    * `rewards_status` -> `status`
* `user_roles` (MAYBE PULL THIS OUT AND USE IN-CODE ABILITIES STYLE)
    * `role_id` -> `id`
    * `role_name` -> `name`
    * `sort_order` (FLOAT -> INT)
* `users`
    * `u_id` -> `id`
    * `rewards_id` -> `reward_group_id`
    * `role_id` -> REMOVE
    * `password_hash`
    * `password_salt`
    * CREATE `encrypted_password` (FOR NEW USERS AND UPDATED PASSWORDS)
    * `name`
    * `email`
    * `terms_conditions_accepted` -> `terms_conditions_accepted_at` (use `created_on`)
    * `password_expired`
    * CREATE `password_last_changed_at`
    * REMOVE `sbm_account`, `src_account`, `grp_account`
    * `created_on` -> timestamps
    * `created_by` -> `created_by_id`


## LEGACY

### Points Calculation

Enrolled customers receive one point per dollar when an invoice is paid full within terms. There are two types of points:

* Earned - all points earned based on the 1-point-per-dollar paid rule.
    - `earned = paid_invoices_total - used`
* Redeemable - all points earned, with a maximum of two times the points earned through Sunroc Building Materials purchases.
    - `redeemable = MAX(earned, 2 * sunroc_building_materials_points)`
*

### Points Expiration

Points do not expire unless the customer hasn't purchased anything from Sunroc within the last two years.
