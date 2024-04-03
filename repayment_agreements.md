MAT GUIDE Section 7 Repayment Agreement notes:

`MOC` **Head Last Name**
`MOC` **Head First Name**

```javascript
IF agreementType EQUALS "T" OR agreementType EQUALS "N" THEN
    // Required for Tenant and No Agreement types
    OUTPUT lastName of head of household based on current or effective certification date
ELSE IF agreementType EQUALS "O" AND unitCount > 1 THEN
    // May be blank if Owner/Agent applies to more than one unit
    OUTPUT blank
END IF
```

`MOC` **Unit Number**

```javascript
IF agreementType EQUALS "T" OR agreementType EQUALS "N" THEN
    // Required for Tenant and No Agreement types
    IF moveOutDate IS NOT NULL THEN
        // If tenant has moved out, use the TRACS unit number at the time of moving out
        OUTPUT tracsUnitNumber as of moveOutDate
    ELSE
        // If tenant has not moved out, use the TRACS unit number as of the first of the month of voucher creation
        OUTPUT tracsUnitNumber as of the first of the month of voucherCreationDate
    END IF
ELSE IF agreementType EQUALS "O" AND unitCount > 1 THEN
    // May be blank if Owner/Agent applies to more than one unit
    OUTPUT blank
END IF
```

`M` **Agreement ID**

```javascript
// Ensure Agreement ID is unique within the project/community
FUNCTION ensureUniqueAgreementId(agreementId)
    IF agreementId is not unique within the project/community THEN
        RETURN error("Agreement ID must be unique")
    END IF
    RETURN true
END FUNCTION

// Handle Agreement ID during software change
FUNCTION transferAgreementIdOnSoftwareChange(originalAgreementId)
    // Assumed logic to transfer the ID; specifics depend on software implementation
    RETURN transferredAgreementId
END FUNCTION

// Use agreement date as ID if unique
FUNCTION useAgreementDateAsID(agreementDate)
    IF ensureUniqueAgreementId(agreementDate) THEN
        RETURN agreementDate
    ELSE
        RETURN generateNewUniqueID()
    END IF
END FUNCTION

// Manage ID for reversing entries not associated with a repayment agreement
IF isReversingEntry AND currentAgreementType EQUALS "N" THEN
    // Record must include an Agreement ID even if reversing and not currently tied to a repayment agreement
    USE currentAgreementId
    // To change into a repayment agreement, send a record with a Total Payment and set Agreement Type to T
    IF totalPaymentReceived AND agreementTypeChangingToT THEN
        SET currentAgreementType = "T"
    END IF
END IF

// Handle renegotiation or new agreement execution
IF isRenegotiated THEN
    // The ID remains constant even if the agreement is renegotiated
    USE currentAgreementId
ELSE IF newAgreementExecuted THEN
    // A new ID must be generated for a completely new agreement
    ASSIGN newUniqueID()
END IF

// Assign a new ID for each instance of misreporting
FOR EACH instance IN misreportingInstances
    ASSIGN newUniqueID() TO instance
    // Note: Even if covered by a single paper agreement, each instance needs a unique ID
NEXT
```

`M` **Agreement Date**

```javascript
// Determine the appropriate date based on conditions
IF agreementStatus EQUALS "signed" THEN
    // Case for transactions associated with a signed repayment agreement
    IF isModified THEN
        // For modified agreements, use the original agreement date
        SET appropriateDate = agreementDate
    ELSE
        // For unmodified agreements, use the agreement date, or tenant signature date if no agreement date
        IF agreementDate IS NOT NULL THEN
            SET appropriateDate = agreementDate
        ELSE
            SET appropriateDate = tenantSignatureDate
        END IF
    END IF
ELSE IF agreementStatus EQUALS "no agreement" THEN
    // Case for reversing transactions not associated with a repayment agreement
    SET appropriateDate = MAX(transactionCreationDate, voucherDate)
END IF

// Additional rule for multiple instances of misreporting under a single paper agreement
IF there are multiple instances of misreporting AND they are covered by a single paper agreement THEN
    // Set the appropriate date to when the paper agreement was revised for new misreporting
    SET appropriateDate = revisedagreementDate
END IF

// Output the determined appropriate date
OUTPUT appropriateDate
```

`M*` **Agreement Amount**

```javascript
// Determine the original agreement amount based on the agreement type
IF agreementType EQUALS "N" THEN
    // For Agreement Type N, use the reversal amount
    SET originalAgreementAmount = amountRequested
ELSE
    // For all other agreement types, use the original agreement amount prior to any payments
    SET originalAgreementAmount = originalAmount
END IF

// Output the determined original agreement amount
OUTPUT originalAgreementAmount
```

`M` **Agreement Type**

```javascript
// Define the process to print the agreement type on the form based on its value
IF agreementType EQUALS "T" THEN
    // If the agreement type is T, print "Tenant" on the form
    PRINT "Tenant"
ELSE IF agreementType EQUALS "O" THEN
    // If the agreement type is O, print "Owner" on the form
    PRINT "Owner"
ELSE IF agreementType EQUALS "N" THEN
    // If the agreement type is N, print "None" on the form
    PRINT "None"
END IF
```

`M*` **Agreement Change Amount**

```javascript
// Define the process to calculate and format the change amount
IF transactionType EQUALS "original reversing entry" THEN
    // For an original reversing entry, the change amount is equal to the agreement amount
    SET changeAmount = agreementAmount
ELSE
    // For other transaction types, use the transaction amount as the change amount
    SET changeAmount = transactionAmount
END IF

// Permit negative values and format them as specified
IF changeAmount < 0 THEN
    // Format negative numbers as right-adjusted, zero-filled negative numbers
    // Example: -575 becomes –000000575
    FORMAT changeAmount AS right-adjusted, zero-filled negative number
    PRINT changeAmount
ELSE
    // Positive values are unsigned and printed as is
    PRINT changeAmount
END IF
```

`M*` **Total Payment**

```javascript
// Initialize totalPayment variable
SET totalPayment = 0

// Determine the Total Payment based on the transaction type and conditions
IF isagreementTypeN THEN
    // For Agreement Type N, Total Payment must always be 0
    totalPayment = 0
ELSE IF transactionType EQUALS "Reversal" THEN
    // For a Reversal, use the lump sum payment amount
    totalPayment = lumpSumPayment
ELSE IF transactionType EQUALS "Tenant Payment" OR transactionType EQUALS "Owner Payment" THEN
    // For tenant or owner payments, use the amount collected this month
    totalPayment = monthlyCollectedAmount
END IF

// Formatting the Total Payment
// Permit negative and format as specified, positive values remain unchanged
IF totalPayment < 0 THEN
    // Convert totalPayment to a right-adjusted, zero-filled negative number, e.g., –000000575
    FORMAT totalPayment AS "right-adjusted, zero-filled negative number"
ELSE
    // Regular payments (positive numbers) are reported as is, unsigned
    // No specific formatting required for positive values
END IF

// Output the formatted Total Payment
OUTPUT totalPayment
```

`M*` **Amount Retained**

```javascript
// Initialize amountRetained variable
SET amountRetained = 0

// Determine the Amount Retained based on transaction type and conditions
IF transactionType EQUALS "Reversal" THEN
    IF lumpSumPayment IS 0 THEN
        // For a Reversal with no Total Payment amount, amount retained is 0
        amountRetained = 0
    ELSE
        // For a Reversal with a lump sum payment, calculate retained expenses
        SET amountRetained = MIN(authorizedExpenses, 0.20 * totalPayment)
    END IF
ELSE IF transactionType EQUALS "Payment" THEN
    // For a Payment, calculate retained expenses
    SET amountRetained = MIN(authorizedExpenses, 0.20 * totalPayment)
END IF

// Formatting the Amount Retained
// Permit negative and format as specified, positive values remain unchanged
IF amountRetained < 0 THEN
    // Convert amountRetained to a right-adjusted, zero-filled negative number, e.g., –000000575
    FORMAT amountRetained AS "right-adjusted, zero-filled negative number"
ELSE
    // The amount related to a regular payment is reported as a positive number
    // No specific formatting required for positive values
END IF

// Output the formatted Amount Retained
OUTPUT amountRetained
```

`M*` **Ending Balance**

```javascript
// Process to calculate the ending balance based on the agreement type
IF agreementType EQUALS "N" THEN
    // For Agreement Type N, the Ending Balance is equal to the Agreement Amount, which is the Amount Requested
    SET endingBalance = amountRequested
ELSE
    // For all other agreement types, calculate the balance as the agreement amount minus the total payments made
    SET endingBalance = agreementAmount - totalPaymentsMade
END IF

// Output the calculated ending balance
OUTPUT endingBalance
```

`M*` **Amount Requested**

```javascript
// Process to determine the transaction amount and format it
IF agreementType EQUALS "N" THEN
    // For Agreement Type N, the Amount Requested is the amount of the reversal
    SET transactionAmount = amountOfReversal
    // Permit negative values and zeros, and format as specified
    IF transactionAmount < 0 THEN
        // Convert to a right-adjusted, zero-filled negative number, e.g., –000000575
        FORMAT transactionAmount AS "right-adjusted, zero-filled negative number"
    ELSE IF transactionAmount IS 0 THEN
        // If the amount is zero, it's allowed and remains as is
        SET transactionAmount = 0
    END IF
ELSE
    // For other agreement types, use the specified transaction amount
    // Negative values are permitted and should be formatted accordingly
    IF transactionAmount < 0 THEN
        FORMAT transactionAmount AS "right-adjusted, zero-filled negative number"
    // Positive values remain as is, unsigned
END IF

// This field contributes to the total in MAT30, Section 2, Field 28
// No additional calculation here, but ensure correct aggregation in Field 28

// Output the formatted transaction amount
OUTPUT transactionAmount
```

`MOC` **Paid Amount**

```javascript
// Process to format the approved amount based on its value and applicability
IF isApplicable THEN
    IF approvedAmount < 0 THEN
        // For negative numbers: sign in the leftmost position, number right-adjusted and zero-filled
        FORMAT approvedAmount AS "negative, right-adjusted, zero-filled"
        // Example: -45 becomes -00045
    ELSE
        // Positive numbers are unsigned and should be left as is
        // Ensure positive numbers are zero-filled as required by site software, if necessary
        FORMAT approvedAmount AS "positive, unsigned"
    END IF
ELSE
    // If the amount is not applicable, set to zeros
    approvedAmount = 0
END IF

// This field contributes to the total in MAT30, Section 2, Field 43
// Ensure that this amount is correctly aggregated or summed in Field 43

// Output the formatted approved amount
OUTPUT approvedAmount
```
