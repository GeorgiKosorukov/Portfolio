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
  )
```
