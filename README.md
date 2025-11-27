-- Select the database
USE fraud_detection;


-- 1. TABLE STRUCTURE (already created earlier)

-- This is the table used in the project

-- CREATE TABLE transactions (
--     step INT,
--     type VARCHAR(20),
--     amount DECIMAL(18, 2),
--     nameOrig VARCHAR(50),
--     oldbalanceOrg DECIMAL(18, 2),
--     newbalanceOrig DECIMAL(18, 2),
--     nameDest VARCHAR(50),
--     oldbalanceDest DECIMAL(18, 2),
--     newbalanceDest DECIMAL(18, 2),
--     isFraud INT,
--     isFlaggedFraud INT
-- );


-- 2. BASIC CHECKS

-- Count total rows
SELECT COUNT(*) AS total_rows
FROM transactions;

-- Look at the first 10 rows
SELECT *
FROM transactions
LIMIT 10;

-- 3. OVERALL FRAUD RATE


SELECT
    COUNT(*) AS total_transactions,
    SUM(CASE WHEN isFraud = 1 THEN 1 ELSE 0 END) AS fraud_transactions,
    ROUND(
        100.0 * SUM(CASE WHEN isFraud = 1 THEN 1 ELSE 0 END) / COUNT(*),
        4
    ) AS fraud_rate_percent
FROM transactions;


-- 4. FRAUD RATE BY TRANSACTION TYPE


SELECT
    type,
    COUNT(*) AS total_txn,
    SUM(CASE WHEN isFraud = 1 THEN 1 ELSE 0 END) AS fraud_txn,
    ROUND(
        100.0 * SUM(CASE WHEN isFraud = 1 THEN 1 ELSE 0 END) / COUNT(*),
        4
    ) AS fraud_rate_percent
FROM transactions
GROUP BY type
ORDER BY fraud_rate_percent DESC;


-- 5. TOP 10 HIGH VALUE FRAUD TRANSACTIONS


SELECT
    step,
    type,
    amount,
    nameOrig,
    oldbalanceOrg,
    newbalanceOrig,
    nameDest,
    oldbalanceDest,
    newbalanceDest
FROM transactions
WHERE isFraud = 1
ORDER BY amount DESC
LIMIT 10;


-- 6. ACCOUNTS TABLE (for JOIN examples)


DROP TABLE IF EXISTS accounts;

CREATE TABLE accounts AS
SELECT DISTINCT
    name AS account_id
FROM (
    SELECT nameOrig AS name FROM transactions
    UNION
    SELECT nameDest AS name FROM transactions
) AS all_names;


-- 7. TRANSACTION TYPES TABLE (lookup)


DROP TABLE IF EXISTS transaction_types;

CREATE TABLE transaction_types AS
SELECT DISTINCT
    type,
    CASE
        WHEN type IN ('TRANSFER', 'CASH_OUT') THEN 'Money moving out'
        WHEN type = 'CASH_IN' THEN 'Money coming in'
        ELSE 'Other'
    END AS type_group
FROM transactions;


-- 8. JOIN EXAMPLE: ADD ACCOUNT INFO TO TRANSACTIONS


SELECT
    t.step,
    t.type,
    t.amount,
    o.account_id AS sender_account,
    d.account_id AS receiver_account,
    t.isFraud
FROM transactions t
JOIN accounts o
    ON t.nameOrig = o.account_id
JOIN accounts d
    ON t.nameDest = d.account_id
LIMIT 20;


-- 9. JOIN + AGGREGATION: FRAUD RATE BY TYPE WITH GROUPING


SELECT
    tt.type,
    tt.type_group,
    COUNT(*) AS total_txn,
    SUM(CASE WHEN t.isFraud = 1 THEN 1 ELSE 0 END) AS fraud_txn,
    ROUND(
        100.0 * SUM(CASE WHEN t.isFraud = 1 THEN 1 ELSE 0 END) / COUNT(*),
        4
    ) AS fraud_rate_percent
FROM transactions t
JOIN transaction_types tt
    ON t.type = tt.type
GROUP BY tt.type, tt.type_group
ORDER BY fraud_rate_percent DESC;


-- 10. RISKY SENDERS: CTE + JOIN


WITH sender_fraud AS (
    SELECT
        nameOrig AS account_id,
        COUNT(*) AS total_sent,
        SUM(CASE WHEN isFraud = 1 THEN 1 ELSE 0 END) AS fraud_sent
    FROM transactions
    GROUP BY nameOrig
)
SELECT
    a.account_id,
    s.total_sent,
    s.fraud_sent,
    ROUND(100.0 * s.fraud_sent / s.total_sent, 2) AS fraud_rate_percent
FROM sender_fraud s
JOIN accounts a
    ON s.account_id = a.account_id
WHERE s.fraud_sent > 0
ORDER BY fraud_rate_percent DESC, s.fraud_sent DESC
LIMIT 20;

