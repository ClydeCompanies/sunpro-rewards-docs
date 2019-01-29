# Sunroc Rewards Program Definitions

## Points

* Redeemable Points
* Eligible Purchases

## Account Status

* "On Hold" - A customer who is not in good standing with Sunroc's Credit Department will be placed on hold. This means their Sunroc Rewards account is accessible as view only. They can not redeem anything because all of their redemption points are on hold. Points never expire, but unpaid invoices continue to accrue negative points.
* "Closed" - A customer who has not purchased anything in two or more years or who is labeled as "closed" by Sunroc will not be able to access their Sunroc Rewards account. The customer's account will be closed and become inaccessible, known as "no access" within user levels. The customer will receive a "This account has been closed" message upon log in. All points will be deleted. Account terms can go directly from "open" to "closed" without ever being on "hold."

## Data Warehouse

### Rewards_Data_View

* `transaction_date` - The date of the transaction. Multiple invoices may have the exact same transaction dates, as can multiple transactions on the same invoice.
* `transaction_type_rewards_invoice` - 1 if this is an invoice, 0 if it is not.
* `transaction_type_rewards_payment` - 1 if this is a payment, 0 if it is not.
* `transaction_line_number` - The line item number on the invoice for the transaction.
* `transaction_contract` - Construction project number
* `transaction_description`
    - Document#(BisTrack)
    - Invoice Number (Vista)
    - Item Reference Code (Command)
* `transaction_amount` - The amount of the transaction. Negative numbers are payments or credits, and positive numbers are invoice amounts.
* `transaction_original_amount` - The original amount before any payments or credits.
* `transaction_retainage` - Only matters for construction (it's like earnest money for a construction project) you don't get points for retainage until it is released.
* `customer_id` - A per-system unique ID of the customer.
* `customer_name` - The human-readable name of the customer.
* `company_abbreviation` - The company where the goods/services were purchased. Possible values are:
    - DDF
    - GRP
    - SBM
    - SCI
    - SRC
    - WWC
* `name` - The human-readable name of the company where the goods/services were purchased. Possible values are:
    - DDF, Inc. - DDF
    - Geneva Rock Products, Inc. - GRP
    - Scott Contracting, Inc. - SCI
    - Sunroc Building Materials, Inc. - SBM
    - Sunroc Corporation - SRC
    - W. W. Clyde & Co. - WWC
* `system_name` - The database from which the data came. Possible values are:
    - BisTrack
    - Command
    - Vista
* `product_line_name` - The name of the product line. Possible values are:
    - Aggregate
    - Asphalt
    - Concrete
    - Construction
    - Custom Doors
    - Framing Components
    - Garage Doors
    - Insulation
    - Lumber
    - Millwork & Doors
    - Other Building Products
    - Trusses
    - Windows
* `customer_master_id` - The unique id of the customer across ALL systems.
