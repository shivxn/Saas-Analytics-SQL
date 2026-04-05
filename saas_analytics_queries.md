# SaaS Analytics Queries

## QUESTION 1: Monthly Recurring Revenue Trend
```sql
SELECT 
    YEAR(start_date) AS year,
    MONTH(start_date) AS month_number,
    MONTHNAME(start_date) AS month_name,
    SUM(mrr_amount) AS total_mrr,
    COUNT(DISTINCT subscription_id) AS active_subscriptions,
    ROUND(AVG(mrr_amount), 2) AS avg_subscription_value
FROM subscriptions
WHERE start_date IS NOT NULL AND mrr_amount > 0
GROUP BY YEAR(start_date), MONTH(start_date), MONTHNAME(start_date)
ORDER BY year, month_number;
```

## QUESTION 2: Customer Churn Analysis
```sql
SELECT 
    reason_code,
    COUNT(*) AS churn_count,
    COUNT(DISTINCT account_id) AS customers_churned,
    ROUND(AVG(refund_amount_usd), 2) AS avg_refund_amount,
    ROUND(COUNT(*) * 100.0 / 
        (SELECT COUNT(*) FROM churn_events), 2) AS churn_percentage
FROM churn_events
WHERE reason_code IS NOT NULL
GROUP BY reason_code
ORDER BY churn_count DESC;
```

## QUESTION 3: Customer Lifetime Value
```sql
SELECT DISTINCT account_id,
    SUM(mrr_amount) AS total_mrr,
    SUM(arr_amount) AS total_arr,
    COUNT(subscription_id) AS total_subscriptions
FROM subscriptions
GROUP BY account_id
ORDER BY total_mrr DESC;
```

## QUESTION 4: Churn Risk Prediction
```sql
SELECT 
    a.account_id,
    a.account_name,
    s.downgrade_flag,
    COUNT(st.ticket_id) AS support_tickets,
    CASE 
        WHEN COUNT(st.ticket_id) > 10 THEN 'High_Risk'
        WHEN COUNT(st.ticket_id) < 5 THEN 'Low_Risk'
        ELSE 'Medium_Risk'
    END AS risk_status
FROM accounts a
JOIN subscriptions s ON a.account_id = s.account_id
LEFT JOIN support_tickets st ON a.account_id = st.account_id
GROUP BY a.account_id, a.account_name, s.mrr_amount, s.downgrade_flag
ORDER BY mrr_amount DESC;
```

## QUESTION 5: Feature Adoption & Engagement
```sql
SELECT DISTINCT feature_name,
    COUNT(DISTINCT usage_count) AS feature_count,
    SUM(usage_duration_secs) AS feature_usage_time_in_sec,
    COUNT(DISTINCT usage_id) AS usg_count,
    COUNT(DISTINCT usage_date) AS day_count,
    ROUND(COUNT(usage_count) * 100 /
        (SELECT COUNT(usage_count) FROM feature_usage), 2) AS feature_percentage
FROM feature_usage
GROUP BY feature_name
ORDER BY feature_count DESC;
```

## QUESTION 6: Support Quality & Impact
```sql
SELECT 
    CASE 
        WHEN first_response_time_minutes <= 60 AND satisfaction_score >= 4.0 
            THEN 'Fast + High Satisfaction'
        WHEN first_response_time_minutes > 120 OR satisfaction_score <= 2.0 
            THEN 'Slow or Low Satisfaction'
        ELSE 'Medium'
    END AS support_experience,
    COUNT(DISTINCT st.account_id) AS total_customers
FROM support_tickets st
LEFT JOIN subscriptions s ON st.account_id = s.account_id
GROUP BY support_experience
ORDER BY total_customers;
```

## QUESTION 7: Subscription Plan Performance
```sql
SELECT DISTINCT plan_tier,
    COUNT(account_id) AS total_customer,
    SUM(mrr_amount) AS total_money,
    ROUND(SUM(mrr_amount) * 100 /
        (SELECT SUM(mrr_amount) FROM subscriptions), 2) AS money_percentage
FROM subscriptions
GROUP BY plan_tier
ORDER BY total_money DESC;
```

## QUESTION 8: Customer Segmentation by Behavior
```sql
SELECT 
    a.account_id,
    a.account_name,
    SUM(s.arr_amount) AS total_spending,
    COUNT(DISTINCT fe.usage_id) AS engagement_count,
    CASE 
        WHEN SUM(s.arr_amount) > 2000000 AND COUNT(DISTINCT fe.usage_id) > 50 THEN 'High Value Active'
        WHEN SUM(s.arr_amount) <= 1000000 AND COUNT(DISTINCT fe.usage_id) < 20 THEN 'Low Risk'
        ELSE 'Mid Tier Risk'
    END AS customer_segment
FROM accounts a
JOIN subscriptions s ON a.account_id = s.account_id
LEFT JOIN feature_usage fe ON s.subscription_id = fe.subscription_id
GROUP BY a.account_id, a.account_name
ORDER BY total_spending DESC;
```

## QUESTION 9: Upgrade & Downgrade Trends
```sql
SELECT
  SUM(CASE WHEN (upgrade_flag IN ('1','true','TRUE','True','yes','Y') OR upgrade_flag = 1 OR upgrade_flag = TRUE) THEN 1 ELSE 0 END) AS upgrade_accounts,
  SUM(CASE WHEN (downgrade_flag IN ('1','true','TRUE','True','yes','Y') OR downgrade_flag = 1 OR downgrade_flag = TRUE) THEN 1 ELSE 0 END) AS downgrade_accounts,
  SUM(CASE WHEN (upgrade_flag IN ('1','true','TRUE','True','yes','Y') OR upgrade_flag = 1 OR upgrade_flag = TRUE) THEN arr_amount ELSE 0 END) AS expansion_arr,
  SUM(CASE WHEN (downgrade_flag IN ('1','true','TRUE','True','yes','Y') OR downgrade_flag = 1 OR downgrade_flag = TRUE) THEN arr_amount ELSE 0 END) AS contraction_arr,
  CASE
    WHEN SUM(CASE WHEN (upgrade_flag IN ('1','true','TRUE','True','yes','Y') OR upgrade_flag = 1 OR upgrade_flag = TRUE) THEN 1 ELSE 0 END)
         > SUM(CASE WHEN (downgrade_flag IN ('1','true','TRUE','True','yes','Y') OR downgrade_flag = 1 OR downgrade_flag = TRUE) THEN 1 ELSE 0 END)
      THEN 'more_accounts_upgrading'
    ELSE 'more_accounts_downgrading'
  END AS account_verdict,
  CASE
    WHEN SUM(CASE WHEN (upgrade_flag IN ('1','true','TRUE','True','yes','Y') OR upgrade_flag = 1 OR upgrade_flag = TRUE) THEN arr_amount ELSE 0 END)
         > SUM(CASE WHEN (downgrade_flag IN ('1','true','TRUE','True','yes','Y') OR downgrade_flag = 1 OR downgrade_flag = TRUE) THEN arr_amount ELSE 0 END)
      THEN 'expansion_higher'
    ELSE 'contraction_higher'
  END AS revenue_verdict
FROM subscriptions;
```

## QUESTION 10: Net Revenue Retention
```sql
SELECT 
    DATE_FORMAT(start_date, '%Y-%m') AS month,
    SUM(mrr_amount) AS current_month_revenue,
    LAG(SUM(mrr_amount)) OVER (ORDER BY DATE_FORMAT(start_date, '%Y-%m')) AS previous_month_revenue
FROM subscriptions
WHERE start_date IS NOT NULL
GROUP BY DATE_FORMAT(start_date, '%Y-%m')
ORDER BY month DESC
LIMIT 2;
```
