# Order Partial Cancellation Design & Implementation

## Background Information

When an order is placed, `BankAccountService` in `perfcommon` creates a single withdrawal transaction in the `user_transaction` table,
updates the balance in the `user_account` table, and deducts points from _active_ deposit transactions. The `withdrawPoints()` method performs
these steps. The order of the active deposit transactions from which points are deducted is determined by the `DeductionTypeCode` property on the `Bank` object
from which the points are withdrawn. The order can be either `FIIFO`, order based on deposit transaction creation;
or `CTEIFO`, order based on deposit transaction expiration.

`BankAccountService` records the affected active deposit transactions _and_ their ordering in the `transaction_transaction_rltnp` table. All affected
active deposit transactions will share the same `user_transaction_id` field value. The value is the primary key of the withdrawal transaction.
The `parent_user_transaction_id` field values reference the primary keys of the affected deposit transactions.

An example of records created in the `transaction_transaction_rltnp` table for a given order is shown in the table below.
Only the relevant columns are shown for clarity.

| transaction_transaction_id | user_transaction_id | parent_user_transaction_id | points_withdrawn |    
| -------------------------: | ------------------: | -------------------------: | ---------------: |
| 8243578                    | 22966035            | 19859079                   | 25               |
| 8243579                    | 22966035            | 20522600                   | 25               |
| 8243580                    | 22966035            | 21069202                   | 50               |
| 8243581                    | 22966035            | 21434905                   | 200              |
| 8243582                    | 22966035            | 21434907                   | 200              |
| 8243583                    | 22966035            | 21530562                   | 200              |
| 8243584                    | 22966035            | 21879877                   | 25               |
| 8243585                    | 22966035            | 21991354                   | 275              |

This order deducted 1,000 points (the sum of all values in the `points_withdrawn` column). The primary key of the withdrawal transaction is `22966035`.
And the primary keys of the deposit transactions are `19859079`, `20522600`, `21069202`, `21434905`, `21434907`, `21530562`, `21879877`, and `21991354`.

## Order Partial Cancellation Requirements & Limitations

Order Partial Cancellation allows a portion or all of the points, deducted from a user's bank account, for a given order be added back to the user's bank account.
To achieve this goal, we adhere to these requirements and limitations:

- Cannot refund (cancel) more points than the points for a given order, less those points already refunded (canceled).
  
  For example if an order deducted 1,000 points and 500 points have already been refunded (canceled), we must not allow more than 500 points be refunded.
  
- No updates to existing records.

  As a consequence, we will create new deposit transactions in the `user_transaction` table instead of updating existing deposit transactions.
  
- The points will be deposited in the reverse order from the order they were deducted.

  For example, if 100 points were deducted from 3 deposit transactions: transaction A, 25 points; transaction B, 25 points; and transaction C, 50 points,
  and we partially canceled 25 points, the 25 points will be added back to transaction C. By that we mean those 25 points will belong to the same program to
  which the points in transaction C belong.

- The points deposited will come from the same program as the points redeemed.

- The expiration date on the points refunded will be extended by the same number of days until the expiration date on the original deposit transaction.

  For example, if a deposit transaction with an expiration date of `7/31/2018` was redeemed on `7/11/2018` and later canceled on `8/11/2018`, the new deposit transaction
  will have an expiration date of `8/31/2018`. This date is computed by taking the difference between the original expiration date and the date of redemption to derive
  the number of days until expiration, which is `20` days. This number is then added to the date of cancellation of `8/11/2018` to derive the new expiration date of `8/31/2018`.
  This calculation is needed to prevent users from gaming the system to "reset" their soon-to-be-expired points.
  
## Order Partial Cancellation API

The Order Partial Cancellation API will consist of an single endpoint that takes these parameters:

- `withdrawTransactionId` - The primary key of the withdraw transaction created when the order is created
- `sourceTransactionId` - A reference ID for the client of the API, for example, the lineItemId. All deposited transactions created will have this value
in their `sourceTransactionId` field.
- `pointsToRefund` - The number of points to be refunded. Must not be greater than the points on the order, less than the points already refunded.

An OpenAPI documentation will be provided separately.

## Order Partial Cancellation Implementation

_The endpoint will be implemented in a separate GitHub project named `perf-redemption-api`._ The project is a Scala Play application. It will have read & write
access to the Victories database tables. Specifically, it needs read & write access to these tables: `user_account`, `user_transaction`, and needs read access
to `transaction_transaction_rltnp`.

The implementation maintains its own tables stored in a Postgres database. Specifically, it maintains two tables: `canceled_order` and `canceled_order_transaction`.
The `canceled_order` table will keep track of which orders have been partially canceled and their remaining cancelable points. Using the `canceled_order`,
implementation can quickly check to see if a client request can be satisfied, i.e., whether there are enough cancelable points remaining.
The `canceled_order_transaction` table will keep track of which deposit transactions have been canceled (refunded) and their remaining cancelable points.
The implementation uses this table to keep track of which deposit transactions can be canceled.

Using the records in the `transaction_transaction_rltnp` for the example order above, here is a complete example on how the implementation uses these tables.

1. Partially cancel 200 points

   The implementation first attempts to look up `withdrawTransactionId 22966035` in the `canceled_order` table, does not find an entry, and
   creates an entry in the  `canceled_order` table:

   | withdraw_transaction_id | order_points_total | points_canceled | points_cancelable |    
   | ----------------------: | -----------------: | --------------: | ----------------: |
   | 22966035                | 1000               | 0               | 1000              |

   The implementation then populates the `canceled_order_transaction` table with these records:

   | user_transaction_id | points_withdrawn | points_canceled | points_cancelable |    
   | ------------------: | ---------------: | --------------: | ----------------: |
   | 21991354            | 275              | 0               | 275               |
   | 21879877            | 25               | 0               | 25                |
   | 21530562            | 200              | 0               | 200               |
   | 21434907            | 200              | 0               | 200               |
   | 21434905            | 200              | 0               | 200               |
   | 21069202            | 50               | 0               | 50                |
   | 20522600            | 25               | 0               | 25                |
   | 19859079            | 25               | 0               | 25                |

   __Note:__ the order of the transactions in `canceled_order_transaction` is the reverse of the order of the transactions in the `transaction_transaction_rltnp` table.

   There are two invariants:

   - The sum of values of the `points_canceled` column in the `canceled_order_transaction` table should equal to the `points_canceled` column in the `canceled_order` table.
   - The sum of values of the `points_cancelable` column in the `canceled_order_transaction` table should equal to the `points_cancelable` column in the `canceled_order` table.

   The implementation creates a new deposit transaction of 200 points in the `user_transaction` table with the `Partial Cancellation` feature set.

   __Note:__ We need to create a new Feature Set with the name `Partial Cancellation`.
 
   The implementation then updates the `canceled_order_transaction` table as follows:

   | user_transaction_id | points_withdrawn | points_canceled | points_cancelable |    
   | ------------------: | ---------------: | --------------: | ----------------: |
   | 21991354            | 275              | 200             | 75                |
   | 21879877            | 25               | 0               | 25                |
   | 21530562            | 200              | 0               | 200               |
   | 21434907            | 200              | 0               | 200               |
   | 21434905            | 200              | 0               | 200               |
   | 21069202            | 50               | 0               | 50                |
   | 20522600            | 25               | 0               | 25                |
   | 19859079            | 25               | 0               | 25                |

   And updates the `canceled_order` table:

   | withdraw_transaction_id | order_points_total | points_canceled | points_cancelable |    
   | ----------------------: | -----------------: | --------------: | ----------------: |
   | 22966035                | 1000               | 200             | 800               |

2. Partially cancel 900 points

   The implementation looks up `withdrawTransactionId 22966035` in the `canceled_order` table and since the value of `points_cancelable` column is `800` (see table above),
   which is less than `900`, we immediately reject the request.

3. Partially cancel 200 points

   The implementation looks up `withdrawTransactionId 22966035` in the `canceled_order` table and the records in the `canceled_order_transaction` table: 

   | user_transaction_id | points_withdrawn | points_canceled | points_cancelable |    
   | ------------------: | ---------------: | --------------: | ----------------: |
   | 21991354            | 275              | 200             | 75                |
   | 21879877            | 25               | 0               | 25                |
   | 21530562            | 200              | 0               | 200               |
   | 21434907            | 200              | 0               | 200               |
   | 21434905            | 200              | 0               | 200               |
   | 21069202            | 50               | 0               | 50                |
   | 20522600            | 25               | 0               | 25                |
   | 19859079            | 25               | 0               | 25                |

   It then creates `3` deposit transactions in the `user_transaction` table based on the Program IDs and Expiration Dates of deposit transactions `21991354`, `21879877`, and `21530562`.

   Updates the records in the `canceled_order_transaction` table:

   | user_transaction_id | points_withdrawn | points_canceled | points_cancelable |    
   | ------------------: | ---------------: | --------------: | ----------------: |
   | 21991354            | 275              | 275             | 0                 |
   | 21879877            | 25               | 25              | 0                 |
   | 21530562            | 200              | 100             | 100               |
   | 21434907            | 200              | 0               | 200               |
   | 21434905            | 200              | 0               | 200               |
   | 21069202            | 50               | 0               | 50                |
   | 20522600            | 25               | 0               | 25                |
   | 19859079            | 25               | 0               | 25                |

   And updates the `canceled_order` table:

   | withdraw_transaction_id | order_points_total | points_canceled | points_cancelable |    
   | ----------------------: | -----------------: | --------------: | ----------------: |
   | 22966035                | 1000               | 400             | 600               |
