1. Partially cancel 200 points

   The implementation first attempts to look up `withdrawTransactionId 22966035` in the `canceled_order` table, does not find an entry, and
   creates an entry in the  `canceled_order` table:

   | withdraw_transaction_id | order_points_total | points_canceled | points_cancelable |    
   | ----------------------: | -----------------: | --------------: | ----------------: |
   | 22966035                | 1000               | 0               | 1000              |
