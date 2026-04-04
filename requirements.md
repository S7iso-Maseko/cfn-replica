# Requirements Document

## Introduction

This document defines the requirements for a hands-on learning project that teaches CloudFormation custom resources through real-world enterprise scenarios. The project covers the lifecycle mechanics of custom resources (Create, Update, Delete), the Lambda-backed and SNS-backed provider models, and the response protocol to the pre-signed S3 URL. Each scenario is grounded in a pattern enterprises commonly use when native CloudFormation resource types are insufficient.

### Enterprise Scenarios Where Custom Resources Are Used

Custom resources fill gaps where CloudFormation's native resource catalog or lifecycle model falls short. Based on CloudFormation architectural patterns and enterprise operational needs, the following categories represent the most common use cases:

1. **Cross-Account and Cross-Service Orchestration** — Enterprises operate multi-account environments (via AWS Organizations, Control Tower). Custom resources enable actions in Account B triggered by a stack in Account A, such as sharing resources, registering in a central catalog, or configuring cross-account IAM trust relationships.

2. **Lookup and Data Fetching at Deploy Time** — Stacks often need values that aren't available as CloudFormation parameters or SSM parameters: the latest AMI ID for a given OS, the current IP ranges for a third-party service, the next available CIDR block from an IPAM system, or configuration from an external CMDB. Custom resources run Lambda functions that query APIs and return data via `Fn::GetAtt`.

3. **Third-Party and External System Integration** — Enterprises integrate with systems outside AWS: DNS providers (e.g., registering records in an on-premises DNS), ITSM platforms (e.g., creating a ServiceNow change request before provisioning), secrets vaults (e.g., HashiCorp Vault), or configuration management databases. Custom resources bridge CloudFormation's lifecycle to these external APIs.

4. **Pre-Flight Validation and Compliance Gating** — Before provisioning resources, enterprises enforce guardrails: checking that a requested CIDR doesn't overlap with existing VPCs, validating that a naming convention is followed, confirming budget approval exists, or verifying that a security review is complete. Custom resources act as validation gates that fail the stack if conditions aren't met.

5. **Post-Provisioning Configuration and Bootstrapping** — Some resources need configuration that CloudFormation doesn't natively support: populating a DynamoDB table with seed data, uploading initial objects to an S3 bucket, running a database migration script, configuring an application after deployment, or warming a cache. Custom resources execute these steps as part of the stack lifecycle.

6. **Resource Cleanup and Lifecycle Management** — On stack deletion, enterprises need cleanup beyond what CloudFormation handles: emptying S3 buckets before deletion (CloudFormation can't delete non-empty buckets), deregistering from service discovery, revoking certificates, cleaning up DNS records, or archiving logs. Custom resources handle teardown logic.

7. **Waiting for Asynchronous Processes** — Some enterprise workflows involve asynchronous steps: waiting for a certificate to be validated, waiting for a database replica to sync, or waiting for an approval in an external system. Custom resources combined with SNS-backed providers or Step Functions can model these wait states.

8. **Generating Derived or Computed Values** — Enterprises need computed values during stack creation: generating unique identifiers, computing hash values, transforming input parameters into derived configurations, or generating random passwords/secrets that are stored in Secrets Manager. Custom resources perform these computations and return results.

## Glossary

- **Custom_Resource**: A CloudFormation resource of type `AWS::CloudFormation::CustomResource` or `Custom::*` that delegates Create, Update, and Delete lifecycle operations to a Lambda function or SNS topic.
- **Provider_Lambda**: The AWS Lambda function that receives custom resource lifecycle events and sends responses to the pre-signed S3 URL.
- **Service_Token**: The ARN of the Lambda function or SNS topic that CloudFormation invokes for custom resource lifecycle events. Must be in the same region as the stack.
- **Response_URL**: The pre-signed S3 URL included in the custom resource event payload, to which the provider must send a SUCCESS or FAILED response.
- **Physical_Resource_ID**: A provider-defined identifier returned in the custom resource response that uniquely identifies the backing resource. Changing this value on Update triggers a Delete of the old resource.
- **Request_Type**: The lifecycle operation type sent by CloudFormation: `Create`, `Update`, or `Delete`.
- **Learning_Project**: The collection of CloudFormation templates, Lambda function code, and documentation that comprise this hands-on tutorial.
- **Scenario_Template**: A CloudFormation template that demonstrates a specific enterprise custom resource pattern.
- **AMI_Lookup_Provider**: A Provider_Lambda that queries the EC2 API to find the latest AMI matching specified filters.
- **S3_Bucket_Emptier**: A Provider_Lambda that deletes all objects and versions from an S3 bucket during stack deletion.
- **Seed_Data_Loader**: A Provider_Lambda that populates a DynamoDB table with initial data during stack creation.
- **Validation_Gate**: A Provider_Lambda that performs pre-flight checks and fails the stack if validation conditions are not met.
- **CIDR_Calculator**: A Provider_Lambda that computes a non-overlapping CIDR block based on existing VPC allocations.
- **Password_Generator**: A Provider_Lambda that generates a random password and stores the value in AWS Secrets Manager.

## Requirements

### Requirement 1: Custom Resource Lifecycle Mechanics

**User Story:** As a cloud engineer, I want to understand the full Create, Update, and Delete lifecycle of custom resources, so that I can build reliable providers that handle all three operations correctly.

#### Acceptance Criteria

1. WHEN a stack containing a Custom_Resource is created, THE Learning_Project SHALL include a Scenario_Template that deploys a Provider_Lambda demonstrating the Create request handling, including parsing the event payload and sending a SUCCESS response to the Response_URL.
2. WHEN a stack containing a Custom_Resource is updated with changed resource properties, THE Learning_Project SHALL include a Scenario_Template that demonstrates the Update request handling, including comparing old and new resource properties from the event payload.
3. WHEN a stack containing a Custom_Resource is deleted, THE Learning_Project SHALL include a Scenario_Template that demonstrates the Delete request handling, including cleanup logic and sending a SUCCESS response to the Response_URL.
4. WHEN a Provider_Lambda fails to send a response to the Response_URL within the ServiceTimeout period, THE Learning_Project SHALL include documentation explaining that CloudFormation marks the resource as CREATE_FAILED and the stack operation fails.
5. WHEN a Provider_Lambda returns a FAILED status in the response, THE Learning_Project SHALL include a Scenario_Template that demonstrates error handling behavior, including the Reason field propagation to stack events.
6. WHEN a Custom_Resource Update response returns a different Physical_Resource_ID than the previous one, THE Learning_Project SHALL include documentation explaining that CloudFormation sends a subsequent Delete request for the old Physical_Resource_ID.

### Requirement 2: Deploy-Time AMI Lookup (Data Fetching Pattern)

**User Story:** As a cloud engineer, I want to look up the latest AMI ID at deploy time using a custom resource, so that I can always launch EC2 instances with the most current image without hardcoding AMI IDs.

#### Acceptance Criteria

1. THE Learning_Project SHALL include a Scenario_Template that deploys an AMI_Lookup_Provider Lambda function and a Custom_Resource that invokes the AMI_Lookup_Provider.
2. WHEN the Custom_Resource receives a Create request, THE AMI_Lookup_Provider SHALL query the EC2 DescribeImages API with filters for owner, name pattern, and architecture passed as resource properties.
3. WHEN the EC2 DescribeImages API returns results, THE AMI_Lookup_Provider SHALL select the most recent image by sorting on CreationDate and return the AMI ID as a response attribute accessible via `Fn::GetAtt`.
4. WHEN the EC2 DescribeImages API returns zero results matching the filters, THE AMI_Lookup_Provider SHALL return a FAILED status with a descriptive Reason indicating no matching AMI was found.
5. THE Scenario_Template SHALL include an EC2 instance resource that references the AMI ID from the Custom_Resource output using `Fn::GetAtt`, demonstrating the data flow from provider to consuming resource.
6. WHEN the Custom_Resource receives a Delete request, THE AMI_Lookup_Provider SHALL return a SUCCESS response without performing any action, because the lookup is a read-only operation with no resources to clean up.

### Requirement 3: S3 Bucket Cleanup on Deletion (Resource Cleanup Pattern)

**User Story:** As a cloud engineer, I want to automatically empty an S3 bucket when a stack is deleted, so that CloudFormation can successfully delete the bucket without manual intervention.

#### Acceptance Criteria

1. THE Learning_Project SHALL include a Scenario_Template that deploys an S3 bucket, an S3_Bucket_Emptier Provider_Lambda, and a Custom_Resource with a DependsOn relationship ensuring the Custom_Resource is deleted before the S3 bucket.
2. WHEN the Custom_Resource receives a Create request, THE S3_Bucket_Emptier SHALL return a SUCCESS response and store the bucket name as the Physical_Resource_ID.
3. WHEN the Custom_Resource receives a Delete request, THE S3_Bucket_Emptier SHALL list all object versions in the bucket identified by the Physical_Resource_ID and delete all objects and versions before returning a SUCCESS response.
4. IF the S3_Bucket_Emptier encounters an AccessDenied error when deleting objects, THEN THE S3_Bucket_Emptier SHALL return a FAILED status with a Reason indicating insufficient permissions.
5. THE Scenario_Template SHALL include an IAM role for the S3_Bucket_Emptier with least-privilege permissions: `s3:ListBucket`, `s3:ListBucketVersions`, `s3:DeleteObject`, and `s3:DeleteObjectVersion` scoped to the specific bucket ARN.
6. WHEN the Custom_Resource receives an Update request, THE S3_Bucket_Emptier SHALL return a SUCCESS response without performing any action, because bucket cleanup only applies during deletion.

### Requirement 4: DynamoDB Seed Data Loading (Post-Provisioning Pattern)

**User Story:** As a cloud engineer, I want to populate a DynamoDB table with initial seed data as part of stack creation, so that the application has the reference data it needs immediately after deployment.

#### Acceptance Criteria

1. THE Learning_Project SHALL include a Scenario_Template that deploys a DynamoDB table, a Seed_Data_Loader Provider_Lambda, and a Custom_Resource that depends on the DynamoDB table.
2. WHEN the Custom_Resource receives a Create request, THE Seed_Data_Loader SHALL write seed data items to the DynamoDB table using the BatchWriteItem API, with the table name and seed data items passed as resource properties.
3. WHEN the BatchWriteItem API returns unprocessed items, THE Seed_Data_Loader SHALL retry the unprocessed items with exponential backoff up to 3 attempts before returning a FAILED status.
4. WHEN the Custom_Resource receives a Delete request, THE Seed_Data_Loader SHALL return a SUCCESS response without deleting the seed data, because the DynamoDB table deletion handles data removal.
5. WHEN the Custom_Resource receives an Update request with changed seed data properties, THE Seed_Data_Loader SHALL write the new seed data items to the DynamoDB table, overwriting existing items with matching keys.
6. THE Scenario_Template SHALL include an IAM role for the Seed_Data_Loader with least-privilege permissions: `dynamodb:BatchWriteItem` scoped to the specific table ARN.

### Requirement 5: Pre-Flight CIDR Validation (Validation Gate Pattern)

**User Story:** As a cloud engineer, I want to validate that a requested VPC CIDR block does not overlap with existing VPCs before provisioning, so that I can prevent network conflicts at deploy time rather than discovering them after resources are created.

#### Acceptance Criteria

1. THE Learning_Project SHALL include a Scenario_Template that deploys a Validation_Gate Provider_Lambda and a Custom_Resource that runs before VPC creation using a DependsOn relationship.
2. WHEN the Custom_Resource receives a Create request, THE Validation_Gate SHALL call the EC2 DescribeVpcs API to retrieve all existing VPC CIDR blocks in the current region.
3. WHEN the requested CIDR block overlaps with any existing VPC CIDR block, THE Validation_Gate SHALL return a FAILED status with a Reason listing the conflicting CIDR blocks.
4. WHEN the requested CIDR block does not overlap with any existing VPC CIDR block, THE Validation_Gate SHALL return a SUCCESS response with the validated CIDR block as a response attribute.
5. THE Scenario_Template SHALL include a VPC resource that depends on the Custom_Resource and uses the validated CIDR from `Fn::GetAtt`, demonstrating the gating pattern where the VPC is only created after validation passes.
6. WHEN the Custom_Resource receives a Delete request, THE Validation_Gate SHALL return a SUCCESS response without performing any action, because validation is a read-only check with no resources to clean up.

### Requirement 6: Random Password Generation with Secrets Manager (Computed Values Pattern)

**User Story:** As a cloud engineer, I want to generate a random password and store it in Secrets Manager as part of stack creation, so that sensitive credentials are never hardcoded in templates or parameters.

#### Acceptance Criteria

1. THE Learning_Project SHALL include a Scenario_Template that deploys a Password_Generator Provider_Lambda, a Custom_Resource, and an AWS Secrets Manager secret.
2. WHEN the Custom_Resource receives a Create request, THE Password_Generator SHALL generate a random password of configurable length (passed as a resource property, defaulting to 32 characters) containing uppercase letters, lowercase letters, digits, and special characters.
3. WHEN the password is generated, THE Password_Generator SHALL store the password in the Secrets Manager secret identified by a secret ARN passed as a resource property, and return the secret ARN as a response attribute.
4. THE Password_Generator SHALL set NoEcho to true in the custom resource response to prevent the password value from appearing in stack events or API responses.
5. WHEN the Custom_Resource receives a Delete request, THE Password_Generator SHALL return a SUCCESS response without deleting the Secrets Manager secret, because the secret has its own DeletionPolicy in the template.
6. WHEN the Custom_Resource receives an Update request with a changed password length property, THE Password_Generator SHALL generate a new password with the updated length and update the Secrets Manager secret value.
7. THE Scenario_Template SHALL include an IAM role for the Password_Generator with least-privilege permissions: `secretsmanager:PutSecretValue` scoped to the specific secret ARN.

### Requirement 7: CIDR Block Calculator (Cross-Service Computation Pattern)

**User Story:** As a cloud engineer, I want to compute the next available non-overlapping CIDR block for a new subnet based on existing allocations, so that I can automate network planning within CloudFormation stacks.

#### Acceptance Criteria

1. THE Learning_Project SHALL include a Scenario_Template that deploys a CIDR_Calculator Provider_Lambda and a Custom_Resource that computes a subnet CIDR within a given VPC CIDR range.
2. WHEN the Custom_Resource receives a Create request with a VPC CIDR and desired subnet prefix length as resource properties, THE CIDR_Calculator SHALL query existing subnets in the VPC and compute the next available CIDR block that does not overlap with existing subnet allocations.
3. WHEN no available CIDR block of the requested prefix length exists within the VPC CIDR range, THE CIDR_Calculator SHALL return a FAILED status with a Reason indicating the VPC CIDR space is exhausted.
4. WHEN a valid CIDR block is computed, THE CIDR_Calculator SHALL return the CIDR block as a response attribute accessible via `Fn::GetAtt`.
5. THE Scenario_Template SHALL include a subnet resource that uses the computed CIDR from the Custom_Resource output, demonstrating how computed values feed into subsequent resource creation.
6. WHEN the Custom_Resource receives a Delete request, THE CIDR_Calculator SHALL return a SUCCESS response without performing any action, because the computation has no side effects to reverse.

### Requirement 8: Provider Lambda Error Handling and Resilience

**User Story:** As a cloud engineer, I want to understand error handling patterns for custom resource providers, so that I can build providers that fail gracefully and do not leave stacks in an unrecoverable state.

#### Acceptance Criteria

1. THE Learning_Project SHALL include documentation and code examples demonstrating that a Provider_Lambda must always send a response to the Response_URL, including in error scenarios, to prevent stack operations from hanging until the ServiceTimeout expires.
2. THE Learning_Project SHALL include a Provider_Lambda code pattern that wraps the handler logic in a try-except block and sends a FAILED response with the exception message as the Reason if any unhandled exception occurs.
3. WHEN a Provider_Lambda is deployed inside a VPC, THE Learning_Project SHALL include documentation explaining that the Lambda function must have a route to the CloudFormation-specific S3 endpoint (via NAT Gateway or VPC endpoint) to send responses to the Response_URL.
4. THE Learning_Project SHALL include a Provider_Lambda code pattern that uses the `cfnresponse` module (for inline Lambda code) or the equivalent manual HTTP PUT to the Response_URL, demonstrating both approaches.
5. IF a Custom_Resource Delete operation fails, THEN THE Learning_Project SHALL include documentation explaining that the stack deletion will fail and the operator must manually resolve the issue or skip the resource using `--retain-resources`.
6. THE Learning_Project SHALL include a Scenario_Template that demonstrates the ServiceTimeout property configuration, showing how to set an appropriate timeout based on the expected duration of the provider operation.

### Requirement 9: Response Protocol and Data Contract

**User Story:** As a cloud engineer, I want to understand the exact data contract between CloudFormation and custom resource providers, so that I can correctly parse incoming events and construct valid responses.

#### Acceptance Criteria

1. THE Learning_Project SHALL include documentation describing the complete event payload structure sent by CloudFormation to the Provider_Lambda, including: RequestType, ServiceToken, ResponseURL, StackId, RequestId, ResourceType, LogicalResourceId, PhysicalResourceId, ResourceProperties, and OldResourceProperties.
2. THE Learning_Project SHALL include documentation describing the required response payload structure: Status (SUCCESS or FAILED), Reason, PhysicalResourceId, StackId, RequestId, LogicalResourceId, NoEcho, and Data.
3. THE Learning_Project SHALL include documentation explaining that the response Data object is limited to 4,096 bytes and that values in the Data object are accessible via `Fn::GetAtt` using the key names.
4. THE Learning_Project SHALL include a Provider_Lambda code example that logs the incoming event payload (with sensitive fields redacted) for debugging purposes.
5. WHEN a Provider_Lambda returns response Data exceeding 4,096 bytes, THE Learning_Project SHALL include documentation explaining that the response is rejected and the stack operation fails.

### Requirement 10: Project Structure and Deployment Guide

**User Story:** As a cloud engineer, I want a well-organized project with clear deployment instructions for each scenario, so that I can work through the tutorials independently and in any order.

#### Acceptance Criteria

1. THE Learning_Project SHALL organize each scenario in a separate subdirectory under `CloudFormation/Custom Resources/` with a naming convention of `scenario-N-description/` (e.g., `scenario-1-lifecycle/`, `scenario-2-ami-lookup/`).
2. THE Learning_Project SHALL include a README.md in each scenario subdirectory containing: the enterprise pattern being demonstrated, prerequisites, deployment commands (create, update, delete), expected outputs, and cleanup instructions.
3. THE Learning_Project SHALL include a top-level README.md in `CloudFormation/Custom Resources/` that lists all scenarios with descriptions, the recommended learning order, and the enterprise pattern category each scenario addresses.
4. THE Learning_Project SHALL include all Provider_Lambda source code inline in the CloudFormation templates using the ZipFile property, so that no external packaging or S3 upload is required for deployment.
5. WHEN a scenario requires specific IAM permissions to deploy, THE Learning_Project SHALL document the required permissions in the scenario README.md.
6. THE Learning_Project SHALL include AWS CLI commands for deploying, updating, and deleting each scenario stack, following the command format established in the existing `CloudFormation/Tutorials/` directory.
