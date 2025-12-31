# Database Schema Recommendations for HMDA CD Waterfall Tagging

## Current Issues Identified

1. **Redundant columns**: `RuleType` (string) vs `RuleTypeId` (FK) - pick one
2. **Missing audit trail tables** for waterfall execution
3. **No rule flow/dependency tracking** for tree-style execution
4. **FieldsToReplace** is a comma-separated string - consider normalization

## Recommended Schema

### Core Tables (Storage Only - No Logic)

```sql
-- ============================================
-- LOOKUP TABLES (Reference Data)
-- ============================================

CREATE TABLE [cap].[ValidationRuleLevel] (
    Id INT PRIMARY KEY IDENTITY(1,1),
    RuleLevelName NVARCHAR(50) NOT NULL,  -- 'Loan', 'Document'
    IsActive BIT NOT NULL DEFAULT 1,
    UpdateDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET()
);

CREATE TABLE [cap].[ValidationRuleType] (
    Id INT PRIMARY KEY IDENTITY(1,1),
    RuleTypeName NVARCHAR(50) NOT NULL,   -- 'data', 'waterfall'
    IsActive BIT NOT NULL DEFAULT 1,
    UpdateDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET()
);

CREATE TABLE [cap].[ValidationRuleSubType] (
    Id INT PRIMARY KEY IDENTITY(1,1),
    RuleSubTypeName NVARCHAR(50) NOT NULL, -- 'Initial', 'Final', 'PCCD'
    IsActive BIT NOT NULL DEFAULT 1,
    UpdateDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET()
);

-- ============================================
-- VALIDATION RULES (Simplified - Remove Redundancy)
-- ============================================

CREATE TABLE [cap].[ValidationRule] (
    Id INT PRIMARY KEY IDENTITY(1,1),
    RuleName NVARCHAR(100) NOT NULL UNIQUE,
    RuleTypeId INT NOT NULL REFERENCES [cap].[ValidationRuleType](Id),
    RuleSubTypeId INT NULL REFERENCES [cap].[ValidationRuleSubType](Id), -- NULL for data rules
    RuleLevelId INT NULL REFERENCES [cap].[ValidationRuleLevel](Id),
    RuleOrder INT NOT NULL DEFAULT 0,
    RuleDescription NVARCHAR(500) NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    CreatedDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET(),
    UpdateDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET(),
    
    INDEX IX_ValidationRule_Type (RuleTypeId, RuleSubTypeId, RuleOrder)
);

-- ============================================
-- RULE FIELD MAPPINGS (Normalized - Replace FieldsToReplace)
-- ============================================

CREATE TABLE [cap].[ValidationRuleField] (
    Id INT PRIMARY KEY IDENTITY(1,1),
    RuleId INT NOT NULL REFERENCES [cap].[ValidationRule](Id),
    FieldId NVARCHAR(20) NOT NULL,        -- e.g., '0532010', '0531888'
    FieldName NVARCHAR(100) NULL,         -- Human-readable name
    FieldRole NVARCHAR(50) NOT NULL,      -- 'Input', 'Output', 'Compare', 'Delta'
    IsActive BIT NOT NULL DEFAULT 1,
    
    INDEX IX_RuleField_Rule (RuleId),
    INDEX IX_RuleField_Field (FieldId)
);

-- ============================================
-- RULE FLOW/DEPENDENCIES (For Tree Execution)
-- ============================================

CREATE TABLE [cap].[ValidationRuleFlow] (
    Id INT PRIMARY KEY IDENTITY(1,1),
    ParentRuleId INT NOT NULL REFERENCES [cap].[ValidationRule](Id),
    ChildRuleId INT NOT NULL REFERENCES [cap].[ValidationRule](Id),
    FlowCondition NVARCHAR(20) NOT NULL,  -- 'Yes', 'No', 'Pass', 'Fail'
    ExecutionOrder INT NOT NULL DEFAULT 0,
    IsActive BIT NOT NULL DEFAULT 1,
    
    CONSTRAINT UQ_RuleFlow UNIQUE (ParentRuleId, ChildRuleId, FlowCondition),
    INDEX IX_RuleFlow_Parent (ParentRuleId),
    INDEX IX_RuleFlow_Child (ChildRuleId)
);

-- ============================================
-- PATH TYPE DEFINITIONS (For Final/Initial/PCCD Path Names)
-- ============================================

CREATE TABLE [cap].[CDPathType] (
    Id INT PRIMARY KEY IDENTITY(1,1),
    PathTypeName NVARCHAR(200) NOT NULL,
    PathTypeCode NVARCHAR(10) NOT NULL,   -- '4a', '5b', '13c', etc.
    RuleSubTypeId INT NOT NULL REFERENCES [cap].[ValidationRuleSubType](Id),
    Description NVARCHAR(500) NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    
    INDEX IX_PathType_SubType (RuleSubTypeId)
);

-- ============================================
-- LOAN PROCESSING TABLES
-- ============================================

CREATE TABLE [cap].[LoanWaterfallRun] (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),
    LoanId INT NOT NULL,
    LoanNumber NVARCHAR(50) NOT NULL,
    RunStartDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET(),
    RunEndDtTm DATETIMEOFFSET NULL,
    Status NVARCHAR(50) NOT NULL,         -- 'Pending', 'Processing', 'Completed', 'ManualReview', 'Error'
    InitialCDId INT NULL,
    FinalCDId INT NULL,
    RequiresManualReview BIT NOT NULL DEFAULT 0,
    ManualReviewReason NVARCHAR(500) NULL,
    LoanLevelBitFlags BIGINT NOT NULL DEFAULT 0,  -- Bit flag storage
    ErrorMessage NVARCHAR(MAX) NULL,
    
    INDEX IX_LoanRun_Loan (LoanId),
    INDEX IX_LoanRun_Status (Status)
);

CREATE TABLE [cap].[ClosingDisclosureResult] (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),
    LoanWaterfallRunId BIGINT NOT NULL REFERENCES [cap].[LoanWaterfallRun](Id),
    CDId INT NOT NULL,
    DocumentRelativePath NVARCHAR(500) NOT NULL,
    TagType NVARCHAR(20) NULL,            -- 'Initial', 'Final', 'PCCD', NULL
    PathTypeId INT NULL REFERENCES [cap].[CDPathType](Id),
    DocumentBitFlags BIGINT NOT NULL DEFAULT 0,  -- Bit flag storage
    ValidationPassed BIT NOT NULL DEFAULT 1,
    
    INDEX IX_CDResult_Run (LoanWaterfallRunId),
    INDEX IX_CDResult_CD (CDId)
);

-- ============================================
-- AUDIT TRAIL (Comprehensive Decision Logging)
-- ============================================

CREATE TABLE [cap].[WaterfallAuditTrail] (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),
    LoanWaterfallRunId BIGINT NOT NULL REFERENCES [cap].[LoanWaterfallRun](Id),
    CDId INT NULL,                        -- NULL for loan-level rules
    RuleId INT NOT NULL REFERENCES [cap].[ValidationRule](Id),
    RuleName NVARCHAR(100) NOT NULL,      -- Denormalized for query performance
    ExecutionOrder INT NOT NULL,
    RuleResult BIT NOT NULL,              -- Pass/Fail
    Decision NVARCHAR(500) NOT NULL,
    BitFlagsBeforeRule BIGINT NULL,
    BitFlagsAfterRule BIGINT NULL,
    ExecutedDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET(),
    
    INDEX IX_Audit_Run (LoanWaterfallRunId),
    INDEX IX_Audit_Rule (RuleId),
    INDEX IX_Audit_Time (ExecutedDtTm)
);

-- ============================================
-- VALIDATION ERROR LOG
-- ============================================

CREATE TABLE [cap].[ValidationErrorLog] (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),
    LoanWaterfallRunId BIGINT NOT NULL REFERENCES [cap].[LoanWaterfallRun](Id),
    CDId INT NOT NULL,
    RuleId INT NOT NULL REFERENCES [cap].[ValidationRule](Id),
    RuleName NVARCHAR(100) NOT NULL,
    FieldId NVARCHAR(20) NULL,
    ExpectedValue DECIMAL(18,4) NULL,
    ActualValue DECIMAL(18,4) NULL,
    DeltaValue DECIMAL(18,4) NULL,
    ErrorMessage NVARCHAR(500) NOT NULL,
    CreatedDtTm DATETIMEOFFSET NOT NULL DEFAULT SYSDATETIMEOFFSET(),
    
    INDEX IX_ValError_Run (LoanWaterfallRunId),
    INDEX IX_ValError_CD (CDId)
);

-- ============================================
-- PCCD TRACKING (Multiple PCCDs per Loan)
-- ============================================

CREATE TABLE [cap].[PCCDResult] (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),
    LoanWaterfallRunId BIGINT NOT NULL REFERENCES [cap].[LoanWaterfallRun](Id),
    CDId INT NOT NULL,
    IssueDate DATE NOT NULL,
    PathTypeId INT NULL REFERENCES [cap].[CDPathType](Id),
    TiebreakGroupId INT NULL,             -- Groups CDs with same issue date
    IsTaggedAsPCCD BIT NOT NULL DEFAULT 0,
    
    INDEX IX_PCCD_Run (LoanWaterfallRunId),
    INDEX IX_PCCD_IssueDate (IssueDate)
);
```

## Key Improvements

### 1. Remove Redundancy
- Removed string `RuleType` column - use `RuleTypeId` FK only
- Single source of truth for type information

### 2. Normalize FieldsToReplace
- New `ValidationRuleField` table
- Each field is a separate row with role designation
- Easier to query and maintain

### 3. Rule Flow for Tree Execution
- `ValidationRuleFlow` captures parent-child relationships
- `FlowCondition` enables Yes/No branching
- C# can load this once and build execution tree in memory

### 4. Comprehensive Audit Trail
- `LoanWaterfallRun` - one row per loan processing attempt
- `WaterfallAuditTrail` - every rule decision logged
- `BitFlagsBeforeRule`/`BitFlagsAfterRule` - snapshot for debugging

### 5. Bit Flags Storage
- Store as `BIGINT` (64 bits) in database
- All flag logic in C#
- DB just stores the computed value

### 6. Path Type Reference
- `CDPathType` table for all path type names
- Matches your flowchart (4a, 5b, 13c, etc.)
- Easy to add new paths without code changes

## Sample Data Inserts

```sql
-- Path Types for Final CD
INSERT INTO [cap].[CDPathType] (PathTypeName, PathTypeCode, RuleSubTypeId, Description) VALUES
('Manual Review for Failing 3 Dates test but signed by Borrower', '1', 2, 'FinalCDPathTypeName = 1'),
('Manual Review for All CDs failing 3 Dates Test and Not signed', '2', 2, 'FinalCDPathTypeName = 2'),
('Latest Issued CD before closing date with no matching issue dates', '3', 2, 'FinalCDPathTypeName = 3'),
('CD Tiebreak: Closing Date matches SI Signing date, latest issue dates match, Key Data Points match, none signed', '4a', 2, NULL),
('CD Tiebreak: Closing Date matches SI Signing date, latest issue dates match, Key Data Points match, only 1 signed', '4b', 2, NULL),
('CD Tiebreak: Closing Date matches SI Signing date, latest issue dates match, Key Data Points match, multiple signed same day', '4c', 2, NULL),
('CD Tiebreak: Closing Date matches SI Signing date, latest issue dates match, Key Data Points match, latest signature', '4d', 2, NULL);

-- Rule Flow Example (3 Dates Test -> Signed Check)
INSERT INTO [cap].[ValidationRuleFlow] (ParentRuleId, ChildRuleId, FlowCondition, ExecutionOrder) VALUES
((SELECT Id FROM [cap].[ValidationRule] WHERE RuleName = '3DatesFinalCDBit'),
 (SELECT Id FROM [cap].[ValidationRule] WHERE RuleName = 'ClosingDateSISigningDateMatchBit'),
 'Pass', 1),
((SELECT Id FROM [cap].[ValidationRule] WHERE RuleName = '3DatesFinalCDBit'),
 (SELECT Id FROM [cap].[ValidationRule] WHERE RuleName = 'SignedCDBit'),
 'Fail', 1);
```

## C# Integration Pattern

The database stores ONLY data. All logic lives in C#:

```csharp
// Load rules once at startup
var rules = await _ruleRepository.GetActiveRulesAsync();
var ruleFlows = await _ruleRepository.GetRuleFlowsAsync();

// Build execution tree in memory
var ruleTree = RuleTreeBuilder.Build(rules, ruleFlows);

// Execute waterfall - all logic in C#
var result = await _waterfallEngine.ExecuteAsync(loan, ruleTree);

// Save results to DB (storage only)
await _resultRepository.SaveRunAsync(result);
await _auditRepository.SaveAuditTrailAsync(result.AuditEntries);
```

## Migration Path

1. Create new tables alongside existing
2. Migrate data from `CapsilonValidationRules` to new `ValidationRule`
3. Parse `FieldsToReplace` into `ValidationRuleField` rows
4. Update C# to use new schema
5. Drop old table
