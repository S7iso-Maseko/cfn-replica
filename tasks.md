# Implementation Plan: Custom Resources Learning

## Overview

Build a hands-on CloudFormation custom resources learning project with 7 enterprise scenarios. Each scenario is a self-contained YAML template with inline Python Lambda code (ZipFile, python3.12), deployed via AWS CLI. Implementation progresses from foundational lifecycle mechanics through increasingly complex patterns, with property-based tests validating correctness throughout.

## Tasks

- [x] 1. Create project structure and top-level README
  - [x] 1.1 Create directory structure for all 7 scenarios under `CloudFormation/Custom Resources/`
    - Create directories: `scenario-1-lifecycle/`, `scenario-2-ami-lookup/`, `scenario-3-s3-cleanup/`, `scenario-4-seed-data/`, `scenario-5-cidr-validation/`, `scenario-6-password-generator/`, `scenario-7-cidr-calculator/`
    - _Requirements: 10.1_
  - [x] 1.2 Create top-level `CloudFormation/Custom Resources/README.md`
    - List all 7 scenarios with descriptions, recommended learning order, and enterprise pattern category
    - Include prerequisites (AWS CLI, IAM permissions, Python 3.12 knowledge)
    - _Requirements: 10.3_
  - [x] 1.3 Set up `tests/` directory with empty test files matching the design test structure
    - Create: `test_ami_lookup.py`, `test_s3_cleanup.py`, `test_seed_data.py`, `test_cidr_validation.py`, `test_password_gen.py`, `test_cidr_calculator.py`, `test_error_handling.py`, `test_event_logging.py`, `test_templates.py`
    - Add `conftest.py` with shared fixtures (sample event payloads, mock context objects)
    - _Requirements: 10.1_

- [ ] 2. Implement Scenario 1 — Lifecycle Mechanics
  - [ ] 2.1 Create `scenario-1-lifecycle/template.yaml`
    - Define `LifecycleProviderRole` with CloudWatch Logs permissions only
    - Define `LifecycleProvider` Lambda (ZipFile, python3.12) that branches on RequestType (Create/Update/Delete)
    - Create handler: log event (redact ResponseURL), generate PhysicalResourceId, return SUCCESS
    - Update handler: compare ResourceProperties vs OldResourceProperties, log diff, return SUCCESS with same PhysicalResourceId
    - Delete handler: log cleanup, return SUCCESS
    - Wrap handler in try-except, send FAILED with exception Reason on error
    - Define `LifecycleResource` as `Custom::Lifecycle` with ServiceToken referencing the Lambda
    - Include `ServiceTimeout` property configuration
    - _Requirements: 1.1, 1.2, 1.3, 1.5, 8.2, 8.4, 8.6, 9.4, 10.4_
  - [ ] 2.2 Create `scenario-1-lifecycle/README.md`
    - Document the lifecycle pattern, Create/Update/Delete behavior
    - Include AWS CLI commands for create-stack, update-stack (with changed properties), delete-stack
    - Document ServiceTimeout behavior and what happens when provider fails to respond
    - Document Physical_Resource_ID change behavior on Update triggering Delete of old resource
    - Document FAILED status and Reason field propagation to stack events
    - Document VPC Lambda routing requirements for Response URL
    - Document `--retain-resources` for failed Delete operations
    - _Requirements: 1.4, 1.5, 1.6, 8.1, 8.3, 8.5, 9.1, 9.2, 9.3, 9.5, 10.2, 10.5, 10.6_

  - [ ]* 2.3 Write property test for error handler response guarantee
    - **Property 11: Error handler always sends a response**
    - **Validates: Requirements 8.2**
  - [ ]* 2.4 Write property test for event logging redaction
    - **Property 12: Event logging redacts sensitive fields**
    - **Validates: Requirements 9.4**

- [ ] 3. Implement Scenario 2 — AMI Lookup (Data Fetching Pattern)
  - [ ] 3.1 Create `scenario-2-ami-lookup/template.yaml`
    - Define `AmiLookupProviderRole` with `ec2:DescribeImages` permission
    - Define `AmiLookupProvider` Lambda (ZipFile, python3.12) that queries EC2 DescribeImages API
    - Create handler: filter by Owner, NamePattern, Architecture from ResourceProperties; sort results by CreationDate descending; return latest AMI ID as `AmiId` in response Data
    - Handle empty results: return FAILED with "No matching AMI found" Reason
    - Delete handler: return SUCCESS without action (read-only)
    - Wrap handler in try-except with FAILED response on error
    - Define `AmiLookup` as `Custom::AmiLookup` with filter parameters
    - Define `DemoInstance` as `AWS::EC2::Instance` using `!GetAtt AmiLookup.AmiId`
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 8.2, 10.4_
  - [ ] 3.2 Create `scenario-2-ami-lookup/README.md`
    - Document the data fetching pattern and AMI lookup use case
    - Include AWS CLI deploy/update/delete commands
    - Document required IAM permissions for deployment
    - _Requirements: 10.2, 10.5, 10.6_
  - [ ]* 3.3 Write property test for AMI selection logic
    - **Property 1: AMI selection picks the most recent image**
    - **Validates: Requirements 2.3**
  - [ ]* 3.4 Write property test for read-only provider Delete behavior (AMI Lookup)
    - **Property 2: Read-only providers return SUCCESS on Delete without side effects**
    - **Validates: Requirements 2.6**

- [ ] 4. Implement Scenario 3 — S3 Bucket Cleanup (Resource Cleanup Pattern)
  - [ ] 4.1 Create `scenario-3-s3-cleanup/template.yaml`
    - Define `DemoBucket` as `AWS::S3::Bucket` with versioning enabled
    - Define `BucketEmptierRole` with `s3:ListBucket`, `s3:ListBucketVersions`, `s3:DeleteObject`, `s3:DeleteObjectVersion` scoped to DemoBucket ARN
    - Define `BucketEmptierProvider` Lambda (ZipFile, python3.12)
    - Create handler: return SUCCESS with BucketName as PhysicalResourceId
    - Update handler: return SUCCESS without action
    - Delete handler: list all object versions, batch-delete all objects and versions, return SUCCESS
    - Handle AccessDenied: return FAILED with permissions error Reason
    - Wrap handler in try-except with FAILED response on error
    - Define `BucketEmptier` as `Custom::BucketEmptier` with `DependsOn: DemoBucket` and BucketName property
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 8.2, 10.4_
  - [ ] 4.2 Create `scenario-3-s3-cleanup/README.md`
    - Document the resource cleanup pattern and deletion ordering
    - Include AWS CLI deploy/delete commands
    - Document required IAM permissions
    - _Requirements: 10.2, 10.5, 10.6_
  - [ ]* 4.3 Write property test for S3 cleanup Create storing bucket name
    - **Property 3: S3 cleanup Create stores bucket name as PhysicalResourceId**
    - **Validates: Requirements 3.2**
  - [ ]* 4.4 Write property test for S3 cleanup Delete removing all versions
    - **Property 4: S3 cleanup Delete removes all object versions**
    - **Validates: Requirements 3.3**

- [ ] 5. Checkpoint — Verify first 3 scenarios
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement Scenario 4 — DynamoDB Seed Data (Post-Provisioning Pattern)
  - [ ] 6.1 Create `scenario-4-seed-data/template.yaml`
    - Define `SeedTable` as `AWS::DynamoDB::Table` with simple key schema
    - Define `SeedDataRole` with `dynamodb:BatchWriteItem` scoped to SeedTable ARN
    - Define `SeedDataProvider` Lambda (ZipFile, python3.12)
    - Create handler: parse SeedItems from ResourceProperties, call BatchWriteItem, retry unprocessed items with exponential backoff (up to 3 attempts), return FAILED if retries exhausted
    - Update handler: write new seed data items (overwrite matching keys)
    - Delete handler: return SUCCESS without action
    - Wrap handler in try-except with FAILED response on error
    - Define `SeedData` as `Custom::SeedData` with `DependsOn: SeedTable`, passing TableName and SeedItems
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 8.2, 10.4_
  - [ ] 6.2 Create `scenario-4-seed-data/README.md`
    - Document the post-provisioning pattern and seed data use case
    - Include AWS CLI deploy/update/delete commands
    - Document required IAM permissions
    - _Requirements: 10.2, 10.5, 10.6_
  - [ ]* 6.3 Write property test for seed data writing
    - **Property 5: Seed data writing on Create and Update**
    - **Validates: Requirements 4.2, 4.5**
  - [ ]* 6.4 Write property test for seed data retry logic
    - **Property 6: Seed data retry on unprocessed items**
    - **Validates: Requirements 4.3**

- [ ] 7. Implement Scenario 5 — CIDR Validation (Validation Gate Pattern)
  - [ ] 7.1 Create `scenario-5-cidr-validation/template.yaml`
    - Define `CidrValidatorRole` with `ec2:DescribeVpcs` permission
    - Define `CidrValidatorProvider` Lambda (ZipFile, python3.12) using `ipaddress` module
    - Create handler: call DescribeVpcs, check requested CIDR against all existing VPC CIDRs for overlap, return FAILED listing conflicting CIDRs if overlap, return SUCCESS with ValidatedCidr if clean
    - Delete handler: return SUCCESS without action
    - Wrap handler in try-except with FAILED response on error
    - Define `CidrValidation` as `Custom::CidrValidation` with RequestedCidr property
    - Define `DemoVpc` as `AWS::EC2::VPC` with `DependsOn: CidrValidation` using `!GetAtt CidrValidation.ValidatedCidr`
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 8.2, 10.4_
  - [ ] 7.2 Create `scenario-5-cidr-validation/README.md`
    - Document the validation gate pattern and CIDR overlap checking
    - Include AWS CLI deploy/delete commands
    - Document required IAM permissions
    - _Requirements: 10.2, 10.5, 10.6_
  - [ ]* 7.3 Write property test for CIDR overlap detection
    - **Property 7: CIDR overlap detection correctness**
    - **Validates: Requirements 5.3, 5.4**

- [ ] 8. Implement Scenario 6 — Password Generator (Computed Values Pattern)
  - [ ] 8.1 Create `scenario-6-password-generator/template.yaml`
    - Define `GeneratedSecret` as `AWS::SecretsManager::Secret` (initially empty/placeholder)
    - Define `PasswordGenRole` with `secretsmanager:PutSecretValue` scoped to GeneratedSecret ARN
    - Define `PasswordGenProvider` Lambda (ZipFile, python3.12) using `secrets` module
    - Create handler: generate random password of configurable length (default 32) with uppercase, lowercase, digits, special chars; call PutSecretValue; return SecretArn in Data with NoEcho=true
    - Update handler (length changed): regenerate password with new length, update secret value
    - Delete handler: return SUCCESS without deleting the secret
    - Wrap handler in try-except with FAILED response on error
    - Define `PasswordGenerator` as `Custom::PasswordGenerator` with PasswordLength and SecretArn properties
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 8.2, 10.4_
  - [ ] 8.2 Create `scenario-6-password-generator/README.md`
    - Document the computed values pattern and password generation use case
    - Include AWS CLI deploy/update/delete commands
    - Document required IAM permissions
    - _Requirements: 10.2, 10.5, 10.6_
  - [ ]* 8.3 Write property test for password generation requirements
    - **Property 8: Password generation meets length and character class requirements**
    - **Validates: Requirements 6.2, 6.6**
  - [ ]* 8.4 Write property test for password response contract
    - **Property 9: Password response includes SecretArn and NoEcho**
    - **Validates: Requirements 6.3, 6.4**

- [ ] 9. Checkpoint — Verify scenarios 4-6
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement Scenario 7 — CIDR Calculator (Cross-Service Computation Pattern)
  - [ ] 10.1 Create `scenario-7-cidr-calculator/template.yaml`
    - Define `CidrCalcRole` with `ec2:DescribeSubnets` permission
    - Define `CidrCalcProvider` Lambda (ZipFile, python3.12) using `ipaddress` module
    - Create handler: query DescribeSubnets for existing subnets in VPC, compute next available CIDR of requested prefix length that doesn't overlap existing allocations, return SubnetCidr in Data; return FAILED if CIDR space exhausted
    - Delete handler: return SUCCESS without action
    - Wrap handler in try-except with FAILED response on error
    - Define `CidrCalculation` as `Custom::CidrCalculator` with VpcId, VpcCidr, SubnetPrefixLength properties
    - Define `DemoSubnet` as `AWS::EC2::Subnet` using `!GetAtt CidrCalculation.SubnetCidr`
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 8.2, 10.4_
  - [ ] 10.2 Create `scenario-7-cidr-calculator/README.md`
    - Document the cross-service computation pattern and CIDR calculation use case
    - Include AWS CLI deploy/delete commands
    - Document required IAM permissions
    - _Requirements: 10.2, 10.5, 10.6_
  - [ ]* 10.3 Write property test for computed subnet CIDR correctness
    - **Property 10: Computed subnet CIDR does not overlap existing subnets**
    - **Validates: Requirements 7.2**

- [ ] 11. Template validation and structural tests
  - [ ]* 11.1 Write property test for inline ZipFile code verification
    - **Property 13: All Lambda functions use inline ZipFile code**
    - **Validates: Requirements 10.4**
  - [ ]* 11.2 Write unit tests for template structure validation
    - Verify each scenario template contains expected resource types, DependsOn relationships, and IAM policies
    - Verify cfn-response usage in all handler code
    - Verify README content in each scenario (deployment commands, prerequisites, cleanup)
    - _Requirements: 10.1, 10.2, 10.4_

- [ ] 12. Final checkpoint — Full test suite and integration
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- All Lambda code is inline (ZipFile) with python3.12 — no external packaging needed
- Each scenario is independently deployable and deletable via AWS CLI
- Property tests use Hypothesis library with minimum 100 iterations per property
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation across scenario groups
