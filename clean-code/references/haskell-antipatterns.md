# Haskell Anti-Patterns Reference

Exhaustive catalogue of principle violations as they manifest in Haskell, with
before/after code. Organised by principle.

---

## 1. Parse Don't Validate

### 1a. String-typed IDs crossing module boundaries

```haskell
-- BAD: raw Text flows everywhere; no way to prevent mixing up IDs
createInvoice :: Text -> Text -> Amount -> IO Invoice
createInvoice merchantId customerId amount = ...

-- GOOD: newtypes are free at runtime, invaluable for correctness
newtype MerchantId = MerchantId { unMerchantId :: Text }
newtype CustomerId = CustomerId { unCustomerId :: Text }

createInvoice :: MerchantId -> CustomerId -> Amount -> IO Invoice
```

### 1b. Validation inside business logic

```haskell
-- BAD: guard chains buried in a handler
processLoan :: LoanRequest -> Flow LoanResponse
processLoan req = do
  when (loanAmount req <= 0) $ throwError "invalid amount"
  when (null (applicantName req)) $ throwError "name required"
  -- actual logic...

-- GOOD: parse at boundary, business logic receives valid type
data ValidLoanRequest = ValidLoanRequest
  { vlrAmount      :: PositiveAmount
  , vlrApplicant   :: NonEmptyText
  }

parseLoanRequest :: LoanRequest -> Either ValidationError ValidLoanRequest
parseLoanRequest req = ValidLoanRequest
  <$> parsePositiveAmount (loanAmount req)
  <*> parseNonEmpty (applicantName req)

processLoan :: ValidLoanRequest -> Flow LoanResponse
processLoan req = ...  -- no guards needed here
```

### 1c. Ignoring Maybe/Either returns

```haskell
-- BAD: caller discards the error signal
_ <- insertMerchant conn merchant
sendEmail merchant  -- proceeds even if insert failed

-- GOOD: sequence explicitly
insertMerchant conn merchant >>= \case
  Left err -> logError err >> throwError (DbError err)
  Right _  -> sendEmail merchant
```

---

## 2. Make Illegal States Unrepresentable

### 2a. Bool fields

```haskell
-- BAD: what does (True, False, True) mean?
data Account = Account
  { isActive   :: Bool
  , isVerified :: Bool
  , isFrozen   :: Bool
  }

-- GOOD: exhaustive, self-documenting
data AccountStatus = Active | PendingVerification | Verified | Frozen
  deriving (Eq, Show, Generic)

data Account = Account { accountStatus :: AccountStatus, ... }
```

### 2b. Stringly-typed enums

```haskell
-- BAD: compiler cannot catch "ACTVE" typo
updateStatus :: Text -> IO ()
updateStatus status
  | status == "ACTIVE" = ...
  | status == "INACTIVE" = ...
  | otherwise = error "unknown status"  -- runtime bomb

-- GOOD
data MerchantStatus = MsActive | MsInactive | MsSuspended
  deriving (Eq, Show, Read, Bounded, Enum, Generic)
```

### 2c. Nested Maybe

```haskell
-- BAD: Maybe (Maybe Email) — what does inner Nothing mean?
data User = User { email :: Maybe (Maybe Email) }

-- GOOD: model the states explicitly
data EmailState
  = EmailNotProvided
  | EmailUnverified Email
  | EmailVerified   Email
```

### 2d. Phantom types for role distinction

```haskell
-- BAD: MerchantId and PartnerId are both Int64 — easy to swap
transferFunds :: Int64 -> Int64 -> Amount -> IO ()

-- GOOD
newtype Id (a :: *) = Id { unId :: Int64 }
  deriving (Eq, Ord, Show)

data Merchant; data Partner

transferFunds :: Id Merchant -> Id Partner -> Amount -> IO ()
```

### 2e. Large tuple returns

```haskell
-- BAD: positional, unnamed, easy to swap
getStats :: IO (Int, Int, Int, Double)

-- GOOD
data MerchantStats = MerchantStats
  { totalTransactions :: Int
  , successCount      :: Int
  , failureCount      :: Int
  , successRate       :: Double
  } deriving (Show, Generic)
```

---

## 3. Deep Modules / SRP

### 3a. Fat typeclass

```haskell
-- BAD: 12 methods, only one impl, leaks implementation detail
class MerchantService m where
  fetchMerchant    :: MerchantId -> m (Maybe Merchant)
  createMerchant   :: CreateReq -> m MerchantId
  updateMerchant   :: MerchantId -> UpdateReq -> m ()
  deleteMerchant   :: MerchantId -> m ()
  listMerchants    :: Pagination -> m [Merchant]
  fetchByEmail     :: Email -> m (Maybe Merchant)
  fetchWithStats   :: MerchantId -> m MerchantWithStats
  sendWelcomeEmail :: MerchantId -> m ()
  ...

-- GOOD: narrow, deep — callers only depend on what they need
data MerchantQueries m = MerchantQueries
  { qFetchMerchant :: MerchantId -> m (Maybe Merchant)
  , qFetchByEmail  :: Email -> m (Maybe Merchant)
  }

data MerchantCommands m = MerchantCommands
  { cCreateMerchant :: CreateReq -> m MerchantId
  , cUpdateMerchant :: MerchantId -> UpdateReq -> m ()
  }
-- email sending is a separate concern entirely
```

### 3b. Function too long / mixed abstraction levels

```haskell
-- BAD: orchestration + validation + transformation + I/O in one function
handleDisbursement :: DisbursementReq -> Handler Response
handleDisbursement req = do
  when (amount req <= 0) $ throwError badRequest
  merchant <- runDb $ fetchMerchant (merchantId req)
  case merchant of
    Nothing -> throwError notFound
    Just m  -> do
      let adjusted = adjustForFees (merchantFeeRate m) (amount req)
      result <- callPaymentGateway (gatewayConfig m) adjusted
      runDb $ insertDisbursement (toDisbursement req result)
      sendNotification (merchantEmail m) result
      pure (toResponse result)

-- GOOD: each function is one abstraction level
handleDisbursement :: DisbursementReq -> Handler Response
handleDisbursement req = do
  validated <- parseOrThrow req
  merchant  <- requireMerchant (vMerchantId validated)
  result    <- processDisbursement merchant validated
  pure (toResponse result)

processDisbursement :: Merchant -> ValidDisbursementReq -> Handler DisbursementResult
processDisbursement merchant req = do
  let adjusted = adjustForFees (merchantFeeRate merchant) (vAmount req)
  result <- callPaymentGateway (gatewayConfig merchant) adjusted
  runDb $ insertDisbursement (toDisbursement req result)
  sendNotification (merchantEmail merchant) result
  pure result
```

---

## 4. DRY

### 4a. Duplicated Beam predicates

```haskell
-- BAD: same filter duplicated in 3 query functions
activeMerchants1 = filter_ (\m -> merchantStatus m ==. val_ "ACTIVE"
                               &&. merchantDeletedAt m ==. nothing_)
activeMerchants2 = filter_ (\m -> merchantStatus m ==. val_ "ACTIVE"
                               &&. merchantDeletedAt m ==. nothing_)

-- GOOD: named predicate, composed
isActiveMerchant :: MerchantT (QExpr Postgres s) -> QExpr Postgres s Bool
isActiveMerchant m = merchantStatus m ==. val_ MsActive
                 &&. merchantDeletedAt m ==. nothing_

activeMerchantsQ = filter_ isActiveMerchant (all_ (merchants db))
```

### 4b. Copy-pasted error construction

```haskell
-- BAD
Left $ ApiError 400 "merchant_not_found" "Merchant not found"
-- ... 8 lines later ...
Left $ ApiError 400 "merchant_not_found" "Merchant not found"

-- GOOD
merchantNotFound :: ApiError
merchantNotFound = ApiError 400 "merchant_not_found" "Merchant not found"
```

---

## 5. YAGNI

### 5a. Typeclass with one instance

```haskell
-- BAD: abstraction for its own sake
class Notifiable a where
  notify :: a -> Message -> IO ()

instance Notifiable SmtpMailer where
  notify mailer msg = sendViaSmtp mailer msg
-- no other instance exists or is planned

-- GOOD: just use the function
notifyViaSmtp :: SmtpMailer -> Message -> IO ()
notifyViaSmtp mailer msg = sendViaSmtp mailer msg
```

### 5b. MTL overconstrained

```haskell
-- BAD: MonadIO + MonadLogger + MonadReader AppEnv + MonadError AppError
-- but function only reads one config field and logs one line
fetchTimeout :: ( MonadIO m, MonadLogger m
                , MonadReader AppEnv m, MonadError AppError m )
             => m NominalDiffTime

-- GOOD: pass what you need
fetchTimeout :: Timeout -> NominalDiffTime
fetchTimeout (Timeout t) = t
-- or if IO truly needed:
fetchTimeout :: MonadReader AppEnv m => m NominalDiffTime
fetchTimeout = asks (configTimeout . appConfig)
```

### 5c. Unused record fields

```haskell
-- BAD: field present but always Nothing or default
data MerchantConfig = MerchantConfig
  { mcWebhookSecret    :: Maybe Text   -- always Nothing in all callers
  , mcRetryPolicy      :: RetryPolicy  -- always defaultRetryPolicy
  , mcFeatureFlags     :: [Text]       -- always []
  }
-- GOOD: delete the fields until they're needed
```

---

## 6. Law of Demeter / SoC

### 6a. God-record threading

```haskell
-- BAD: function drills into AppEnv to get one field
sendAlert :: AppEnv -> Alert -> IO ()
sendAlert env alert = do
  let key = slackApiKey (notificationConfig (appConfig env))
  postToSlack key alert

-- GOOD: pass only what you need
sendAlert :: SlackApiKey -> Alert -> IO ()
sendAlert key alert = postToSlack key alert
```

### 6b. Domain logic importing infrastructure

```haskell
-- BAD: domain module knows about DB
module Domain.Merchant where
import qualified Database.Beam.Postgres as Pg  -- NO

processMerchant :: Pg.Connection -> MerchantId -> IO ()

-- GOOD: domain only knows about its algebra
module Domain.Merchant where

data MerchantRepo m = MerchantRepo
  { repoFetch  :: MerchantId -> m (Maybe Merchant)
  , repoUpsert :: Merchant -> m ()
  }

processMerchant :: Monad m => MerchantRepo m -> MerchantId -> m ()
```

---

## 7. Explicit over Implicit

### 7a. Bool parameters

```haskell
-- BAD: what does True mean here?
fetchTransactions :: MerchantId -> Bool -> Bool -> IO [Transaction]
fetchTransactions mid True False = -- includeRefunds=T, pendingOnly=F

-- GOOD
data TransactionFilter = TransactionFilter
  { tfIncludeRefunds :: Bool
  , tfPendingOnly    :: Bool
  }

fetchTransactions :: MerchantId -> TransactionFilter -> IO [Transaction]
```

### 7b. Partial functions

```haskell
-- BAD
getFirstTransaction :: [Transaction] -> Transaction
getFirstTransaction = head  -- crashes on []

-- GOOD
getFirstTransaction :: NonEmpty Transaction -> Transaction
getFirstTransaction = NE.head
-- or if emptiness is possible:
getFirstTransaction :: [Transaction] -> Maybe Transaction
getFirstTransaction = listToMaybe
```

### 7c. Implicit ordering via IO sequencing

```haskell
-- BAD: must call initCache before fetchMerchant but nothing enforces it
initCache :: IO ()
fetchMerchant :: MerchantId -> IO (Maybe Merchant)  -- crashes if not inited

-- GOOD: make the dependency explicit in types
newtype Cache = Cache { unCache :: IORef (Map MerchantId Merchant) }

initCache :: IO Cache
fetchMerchant :: Cache -> MerchantId -> IO (Maybe Merchant)
```

---

## 8. Composition over Inheritance

### 8a. Newtype-deriving overreach

```haskell
-- BAD: deriving Num for a money type — you don't want (*) on Money
newtype Amount = Amount { unAmount :: Decimal }
  deriving (Num, Fractional)  -- (* amount otherAmount) makes no sense

-- GOOD: derive only what makes semantic sense
newtype Amount = Amount { unAmount :: Decimal }
  deriving (Eq, Ord, Show, Generic)

addAmounts :: Amount -> Amount -> Amount
addAmounts (Amount a) (Amount b) = Amount (a + b)

scaleAmount :: Rational -> Amount -> Amount
scaleAmount r (Amount a) = Amount (fromRational r * a)
```

---

## 9. Boy Scout / Dead Code

```haskell
-- Check for:
-- 1. Functions in .hs file not mentioned in the module's export list
--    AND not in any visible import elsewhere
-- 2. `where` clauses that are defined but not used in the body
-- 3. Imports with explicit lists where some symbols are unused:
--    import Data.Map (fromList, toList, empty)  -- if `empty` unused, remove it
-- 4. Language pragmas that GHC -Wall would warn about (-Wunused-imports,
--    -Wunused-matches, -Wmissing-signatures)
```

---

## 10. Fail Fast

```haskell
-- BAD: error propagated silently, fails at consumer
fetchAndProcess :: MerchantId -> IO Result
fetchAndProcess mid = do
  merchant <- fetchMerchant mid  -- returns Nothing on failure
  let m = fromMaybe defaultMerchant merchant  -- silently uses wrong default
  process m

-- GOOD: fail immediately at the site of missing data
fetchAndProcess :: MerchantId -> IO (Either AppError Result)
fetchAndProcess mid = do
  fetchMerchant mid >>= \case
    Nothing       -> pure (Left (MerchantNotFound mid))
    Just merchant -> Right <$> process merchant
```