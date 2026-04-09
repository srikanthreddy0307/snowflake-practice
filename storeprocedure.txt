CREATE DATABASE IF NOT EXISTS PUBLIC;
USE DATABASE PUBLIC;
USE SCHEMA PUBLIC;

CREATE OR REPLACE TABLE PUBLIC.PUBLIC.CUSTOMER_BANK_ACCOUNT (
    ACCOUNT_ID INT AUTOINCREMENT PRIMARY KEY,
    CUSTOMER_NAME VARCHAR(100) NOT NULL,
    ACCOUNT_NUMBER VARCHAR(20) NOT NULL UNIQUE,
    ACCOUNT_TYPE VARCHAR(20) DEFAULT 'SAVINGS',
    BALANCE DECIMAL(15,2) DEFAULT 0.00,
    STATUS VARCHAR(10) DEFAULT 'ACTIVE',
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    UPDATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

CREATE OR REPLACE PROCEDURE PUBLIC.PUBLIC.MANAGE_CUSTOMER_BANK_ACCOUNT(
    ACTION VARCHAR,
    P_ACCOUNT_ID INT DEFAULT NULL,
    P_CUSTOMER_NAME VARCHAR DEFAULT NULL,
    P_ACCOUNT_NUMBER VARCHAR DEFAULT NULL,
    P_ACCOUNT_TYPE VARCHAR DEFAULT 'SAVINGS',
    P_AMOUNT DECIMAL DEFAULT NULL
)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    CASE UPPER(ACTION)
        WHEN 'ADD' THEN
            INSERT INTO PUBLIC.PUBLIC.CUSTOMER_BANK_ACCOUNT (CUSTOMER_NAME, ACCOUNT_NUMBER, ACCOUNT_TYPE, BALANCE)
            VALUES (:P_CUSTOMER_NAME, :P_ACCOUNT_NUMBER, :P_ACCOUNT_TYPE, COALESCE(:P_AMOUNT, 0));
            RETURN 'Account created successfully for ' || :P_CUSTOMER_NAME;

        WHEN 'GET' THEN
            LET result VARCHAR := '';
            SELECT CUSTOMER_NAME || ' | ' || ACCOUNT_NUMBER || ' | ' || ACCOUNT_TYPE || ' | Balance: ' || BALANCE || ' | Status: ' || STATUS
            INTO :result
            FROM PUBLIC.PUBLIC.CUSTOMER_BANK_ACCOUNT
            WHERE ACCOUNT_ID = :P_ACCOUNT_ID;
            RETURN :result;

        WHEN 'DEPOSIT' THEN
            UPDATE PUBLIC.PUBLIC.CUSTOMER_BANK_ACCOUNT
            SET BALANCE = BALANCE + :P_AMOUNT, UPDATED_AT = CURRENT_TIMESTAMP()
            WHERE ACCOUNT_ID = :P_ACCOUNT_ID AND STATUS = 'ACTIVE';
            RETURN 'Deposited ' || :P_AMOUNT || ' to account ' || :P_ACCOUNT_ID;

        WHEN 'WITHDRAW' THEN
            LET current_balance DECIMAL(15,2) := 0;
            SELECT BALANCE INTO :current_balance
            FROM PUBLIC.PUBLIC.CUSTOMER_BANK_ACCOUNT
            WHERE ACCOUNT_ID = :P_ACCOUNT_ID AND STATUS = 'ACTIVE';
            IF (:current_balance >= :P_AMOUNT) THEN
                UPDATE PUBLIC.PUBLIC.CUSTOMER_BANK_ACCOUNT
                SET BALANCE = BALANCE - :P_AMOUNT, UPDATED_AT = CURRENT_TIMESTAMP()
                WHERE ACCOUNT_ID = :P_ACCOUNT_ID;
                RETURN 'Withdrew ' || :P_AMOUNT || ' from account ' || :P_ACCOUNT_ID;
            ELSE
                RETURN 'Insufficient balance. Current balance: ' || :current_balance;
            END IF;

        WHEN 'CLOSE' THEN
            UPDATE PUBLIC.PUBLIC.CUSTOMER_BANK_ACCOUNT
            SET STATUS = 'CLOSED', UPDATED_AT = CURRENT_TIMESTAMP()
            WHERE ACCOUNT_ID = :P_ACCOUNT_ID;
            RETURN 'Account ' || :P_ACCOUNT_ID || ' closed successfully';

        ELSE
            RETURN 'Invalid action. Use: ADD, GET, DEPOSIT, WITHDRAW, or CLOSE';
    END CASE;
END;
$$;

-- --------------------------------------------------------------
CREATE DATABASE IF NOT EXISTS BANK;

CREATE SCHEMA IF NOT EXISTS CUSTOMER;
USE DATABASE BANK;
USE SCHEMA CUSTOMER;

-- Create ATM Transaction table
CREATE OR REPLACE TABLE ATM_TRANSACTIONS (
    transaction_id NUMBER AUTOINCREMENT,
    account_number VARCHAR(20),
    transaction_type VARCHAR(20),
    amount DECIMAL(10,2),
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Create Bank Account table
CREATE OR REPLACE TABLE BANK_ACCOUNTS (
    account_number VARCHAR(20),
    account_holder VARCHAR(100),
    balance DECIMAL(10,2),
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Insert sample account

CREATE OR REPLACE PROCEDURE INSERT_BANK_ACCOUNT(
    account_number VARCHAR,
    account_holder VARCHAR,
    balance FLOAT
)
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    INSERT INTO BANK_ACCOUNTS (account_number, account_holder, balance)
    VALUES (:account_number, :account_holder, :balance);
    RETURN 'Account successfully created';
END;
$$;

-- Create procedure for deposit
CREATE OR REPLACE PROCEDURE DEPOSIT(account_num VARCHAR, deposit_amount DECIMAL)
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    UPDATE BANK_ACCOUNTS 
    SET balance = balance + :deposit_amount 
    WHERE account_number = :account_num;
    
    INSERT INTO ATM_TRANSACTIONS (account_number, transaction_type, amount)
    VALUES (:account_num, 'DEPOSIT', :deposit_amount);
    
    RETURN 'Deposit successful';
END;
$$;

-- Create procedure for withdrawal
CREATE OR REPLACE PROCEDURE WITHDRAW(account_num VARCHAR, withdraw_amount DECIMAL)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    current_balance DECIMAL;
BEGIN
    SELECT balance INTO :current_balance 
    FROM BANK_ACCOUNTS 
    WHERE account_number = :account_num;
    
    IF (current_balance >= :withdraw_amount) THEN
        UPDATE BANK_ACCOUNTS 
        SET balance = balance - :withdraw_amount 
        WHERE account_number = :account_num;
        
        INSERT INTO ATM_TRANSACTIONS (account_number, transaction_type, amount)
        VALUES (:account_num, 'WITHDRAWAL', :withdraw_amount);
        
        RETURN 'Withdrawal successful';
    ELSE
        RETURN 'Insufficient funds';
    END IF;
END;
$$;

-- Create procedure for balance check
CREATE OR REPLACE PROCEDURE CHECK_BALANCE(account_num VARCHAR)
RETURNS DECIMAL
LANGUAGE SQL
AS
$$
DECLARE
    current_balance DECIMAL;
BEGIN
    SELECT balance INTO :current_balance 
    FROM BANK_ACCOUNTS 
    WHERE account_number = :account_num;
    
    RETURN current_balance;
END;
$$;



CALL INSERT_BANK_ACCOUNT('123456789', 'John Doe', 1000.00);
CALL INSERT_BANK_ACCOUNT('ACC001', 'John Doe', 1000.00);
CALL DEPOSIT('ACC001', 500.00);
CALL WITHDRAW('ACC001', 100.00);
CALL CHECK_BALANCE('ACC001');