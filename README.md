# Portfolio

``` sql
SELECT distinct
  p.created_at,
  p.invoice_id,
  p.party_id,
  p.shop_id,
  p.amount,
  p.currency_code,
  pi.email,
  si.status,
  pi.payment_tool_type,
  CONCAT(pi.bank_card_bin, 'XXXX', pi.bank_card_masked_pan) AS card_mask
FROM 
  payment p
JOIN payment_info pi ON pi.invoice_id = p.invoice_id
JOIN status_info  si ON si.invoice_id = p.invoice_id 
WHERE TRUE
AND p.created_at >= NOW() - INTERVAL '120d'
AND si.status = 'success'
AND pi.payment_type = 'bank_card'
AND NOT EXISTS (
   SELECT 1
   FROM payment p_recent
   JOIN payment_info pi_recent ON pi_recent.id = p_recent.id 
   WHERE p_recent.event_created_at >= NOW() - INTERVAL '30d' 
   AND pi_recent.email = pi.email 
  );
```


``` sql
WITH CustomersLists AS (
    SELECT trim(regexp_split_to_table('{{default_email}}', '[ ,]+')) AS default_email
)


SELECT 
    default_email,
    SUM(cost) AS total_sum,
    currency,
    SUM(CASE WHEN type = 'chargeback' THEN cost ELSE 0 END) AS total_chargeback_sum,
    SUM(CASE WHEN type = 'chargeback' THEN 1 ELSE 0 END) AS count_chargeback,
    (SUM(CASE WHEN type = 'chargeback' THEN cost ELSE 0 END) * 100)
    / NULLIF(SUM(cost),0) AS chargeback_procentage
FROM  transaction
WHERE
    ('{{default_email}}' = '%' OR default_email IN (SELECT default_email FROM CustomersLists))
AND 
    payed at time zone 'MSK' BETWEEN to_date('{{ date_range.start }}', 'YYYY-MM-DD') 
AND 
    to_date('{{ date_range.end }}'  ' 23:59:59', 'YYYY-MM-DD HH24:MI:SS') + interval '1 day'
AND 
    payed BETWEEN to_date('{{ date_range.start }}', 'YYYY-MM-DD') - interval '5 hours' 
AND 
    to_date('{{ date_range.end }}'  ' 23:59:59', 'YYYY-MM-DD HH24:MI:SS') + interval '1 day'
GROUP BY 
    default_email, currency
```

``` sql
WITH 
TransactionLists AS (
    SELECT
        f1.service_id,
        f1.currency,
        f1.customer_id,
        f1.customer_email,
        f1.customer_phone,
        f1.customer_ip,
        f1.card_hash,
        SUM(CASE WHEN f1.status = 'success' THEN f1.amount ELSE NULL END) as total_amount,
        SUM(CASE WHEN f1.status = 'success' THEN 1 ELSE 0 END) as count_success,  
        SUM(CASE WHEN f1.status = 'decline' THEN 1 ELSE 0 END) as count_decline 
    FROM 
        transaction_transaction f1
    WHERE 
        f1.create_time >= NOW() - INTERVAL '14d' - INTERVAL '3h'
    GROUP BY 
         f1.service_id, f1.currency, f1.customer_id, f1.customer_email, f1.customer_phone, f1.customer_ip, f1.card_hash
),

CustomersLists AS (
    SELECT trim(regexp_split_to_table('{{customer_id}}', '[ ,]+')) AS customer_id
),

ServiceLists AS (
    SELECT trim(regexp_split_to_table('{{service_id}}', '[ ,]+')) AS service_id
),

EmailLists AS (
    SELECT trim(regexp_split_to_table('{{customer_email}}', '[ ,]+')) AS customer_email
),

CustomerPhone AS (
    SELECT trim(regexp_split_to_table('{{customer_phone}}', '[ ,]+')) AS customer_phone
),

CustomerIP AS (
    SELECT trim(regexp_split_to_table('{{customer_ip}}', '[ ,]+'))::inet AS customer_ip
),

CardHash AS (
    SELECT trim(regexp_split_to_table('{{card_hash}}', '[ ,]+')) AS card_hash
)

SELECT
    tl.customer_id,
    tl.customer_email,
    total_amount,
    tl.currency,
    tl.count_success, 
    tl.count_decline,
    tl.customer_phone,
    tl.customer_ip,
    tl.card_hash
FROM 
    TransactionLists tl
JOIN  
    transa_transa tta  
ON 
    tta.customer_id = tl.customer_id
WHERE 
    ('{{customer_id}}' = '%' OR tl.customer_id IN (SELECT customer_id FROM CustomersLists))
AND
    ('{{service_id}}'= '%' OR tl.service_id IN (SELECT service_id::bigint FROM ServiceLists))
AND
    ('{{customer_email}}'= '%' OR tl.customer_email IN (SELECT customer_email FROM EmailLists))
AND
    ('{{customer_phone}}'= '%' OR tl.customer_phone IN (SELECT customer_phone FROM CustomerPhone))
AND
    ('{{customer_ip}}'= '%' OR tl.customer_ip = ANY (SELECT customer_ip FROM CustomerIP))
AND 
    ('{{card_hash}}'= '%' OR tl.card_hash IN (SELECT card_hash FROM CardHash))
GROUP BY 
     tl.customer_id, tl.customer_email, total_amount, tl.currency, tl.count_success, tl.count_decline, tl.customer_phone, tl.customer_ip, tl.card_hash
LIMIT 1000;
```
