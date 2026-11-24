# Direct Debit Orchestration Service

## 1. BatchRegistrationTaskDefinition

### Purpose

Registers the batch in PSN archival service.

### Input

- GBID

### Calls

- PSNDetailsProcessServiceClient.registerBatch()

### Success

- Stores ARCH_SEQ_ID
- Audit: BATCH_REGISTERED_SUCCESSFULLY

### Failure

- Missing GBID = NonRetryable
- RetryableIntegrationException = workflow suspension
- Unexpected exception = NonRetryable

### Variables Updated

- ARCH_SEQ_ID

## 2. FetchFileChannelDetailsTaskDefinition

### Purpose

Fetches creditor and file channel information from payment-process-service.

### Calls

`paymentProcessServiceClient.getBatchDetails(gbid)`

### Output

Stores:

- Creditor IBAN
- Creditor Name
- Ultimate Creditor
- Address details

### Variable Written

- FILE_CHANNEL_DETAILS

### Failure Handling

| Scenario | Action |
|---|---|
| PPS011 | NonRetryable |
| 401 Unauthorized | NonRetryable |
| Any other error | Retryable |

## 3. AccountActiveCheckTaskDefinition

### Purpose

Validates whether creditor account is active.

### Inputs

- Customer ID
- Creditor IBAN

### Call

`accountProcessServiceClient.getAccountPreferences()`

### Success Logic

| Account Status | Action |
|---|---|
| Active | IS_ACCOUNT_ACTIVE=true |
| Not Active | IS_ACCOUNT_ACTIVE=false<br>PSN_IN_REASON_CODE=CreditorAccountInactive |

### Additional Data Stored

- SWIFT_ADDRESS

### Special Handling

If API returns inReasonCode:

- Sets account inactive
- Stores PSN reason code
- Avoids workflow failure
## 4. TranslateToRpfTaskDefinition

### Purpose

Triggers Batch Converter service.

### Action

Publishes MQ request:

- CustId
- GBID

to `bdd-ms-batch-converter-to-file-service`

### Result

Async processing begins.

## 5. TranslationToRpfResponseTaskDefinition

### Purpose

Receives response from Batch Converter.

### Response Contains

- File Path
- File Size
- Transaction Count

### Logic

| Transaction Count | Outcome |
|---|---|
| 0 | IS_VALID_TRANSACTIONS=false<br>BATCH_TRANSLATION_ZERO_RECORDS |
| >0 | IS_VALID_TRANSACTIONS=true<br>Store converter response |

## 6. FetchRoutingPreferenceTaskDefinition

### Purpose

Determines routing destination.

### Key Decision Inputs

- Brand
- Transaction Count
- Routing Indicator

### Brand Resolution

| SWIFT Prefix | Brand |
|---|---|
| NWBK | NWB |
| RBOS | RBS |

If unavailable:

- Calls customer-process-service

### Routing Size Determination

| Condition | Indicator |
|---|---|
| Txn Count > configured limit | L |
| Else | S |

### Output Variables

- SRT_WRAPPER_QUEUE_NAME
- SRT_WRAPPER_ROUTE_FOUND
- IS_ZIPPING_REQUIRED

### Route Not Found

If SPS117:

- Route set null
- PSN_IN_REASON_CODE=PaymentRouteNotFound