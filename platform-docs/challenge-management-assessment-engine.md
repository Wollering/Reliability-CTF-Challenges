# Challenge Management and Assessment Engine Capabilities

This document provides detailed information about the Challenge Management and Assessment Engine capabilities of the Dynamic Challenge System for AWS Reliability CTF.

## Table of Contents

1. [Challenge Management Capabilities](#challenge-management-capabilities)
   - [Challenge Storage and Structure](#challenge-storage-and-structure)
   - [Challenge Registry](#challenge-registry)
   - [Challenge Configuration Format](#challenge-configuration-format)
   - [Challenge Management API](#challenge-management-api)
   - [Challenge Developer Workflow](#challenge-developer-workflow)
2. [Assessment Engine Capabilities](#assessment-engine-capabilities)
   - [Dynamic Assessment Process](#dynamic-assessment-process)
   - [Dynamic Code Loading](#dynamic-code-loading)
   - [Cross-Account Assessment](#cross-account-assessment)
   - [Check Function Interface](#check-function-interface)
   - [Scoring and Feedback Generation](#scoring-and-feedback-generation)
   - [Security Measures](#security-measures)
3. [Integration Between Components](#integration-between-components)

## Challenge Management Capabilities

### Challenge Storage and Structure

The system stores challenges in a dedicated S3 bucket with a standardized directory structure:

```
s3://ctf-reliability-challenges/
├── voting-system/
│   ├── config.json               # Challenge configuration
│   ├── check-functions.js        # Assessment logic
│   └── resources/                # Challenge-specific resources
├── api-service/
│   ├── config.json
│   ├── check-functions.js
│   └── resources/
```

Each challenge follows a consistent pattern:
- `config.json` defines metadata, assessment criteria, and scoring
- `check-functions.js` contains the actual code to evaluate solutions
- `resources/` directory holds reference materials and assets

### Challenge Registry

Challenges are registered in a DynamoDB table that serves as the central catalog:

```json
{
  "challengeId": "reliability-voting-system",
  "name": "Reliable Voting System",
  "description": "Improve a voting system's reliability during peak traffic",
  "s3Location": "s3://ctf-reliability-challenges/voting-system/",
  "configFile": "config.json",
  "checkFunctionsFile": "check-functions.js",
  "difficulty": "intermediate",
  "active": true,
  "createdBy": "engineer@example.com",
  "createdAt": "2023-05-15T14:30:00Z"
}
```

This registry enables:
- Discoverability of available challenges
- Filtering by difficulty, status, or topic
- Versioning of challenge definitions
- Activation/deactivation without removing challenges

### Challenge Configuration Format

The `config.json` file defines assessment criteria in a structured format:

```json
{
  "challengeId": "voting-system",
  "name": "Reliable Voting System",
  "description": "Improve a voting system experiencing reliability issues during peak traffic",
  "assessmentCriteria": [
    {
      "id": "multi-region",
      "name": "Multi-Region Deployment",
      "points": 10,
      "checkFunction": "checkMultiRegionDeployment",
      "description": "Deploy resources across multiple AWS regions",
      "suggestion": "Consider using DynamoDB Global Tables and multi-region Lambda deployments"
    },
    {
      "id": "dynamodb-backup",
      "name": "DynamoDB Point-in-Time Recovery",
      "points": 10, 
      "checkFunction": "checkDynamoDBBackups",
      "description": "Enable point-in-time recovery for data protection",
      "suggestion": "Configure continuous backups with PITR enabled"
    }
  ],
  "stackNamePrefix": "reliability-challenge-",
  "passingScore": 80
}
```

This configuration maps assessment criteria to specific check functions and assigns point values.

### Challenge Management API

The API Gateway exposes RESTful endpoints for challenge administration:

- **GET /challenges** - Lists available challenges with filtering options
- **GET /challenges/{id}** - Retrieves detailed challenge information
- **POST /challenges** - Registers a new challenge
- **PUT /challenges/{id}** - Updates an existing challenge
- **DELETE /challenges/{id}** - Removes a challenge from the registry
- **PUT /challenges/{id}/activate** - Activates a challenge
- **PUT /challenges/{id}/deactivate** - Deactivates a challenge

All endpoints are protected by Cognito authentication with administrator privileges.

### Challenge Developer Workflow

The system supports this workflow for challenge developers:

1. Develop check functions locally using the standardized interface
2. Test functions against sample resources
3. Package challenge files (config, check functions, resources)
4. Upload to the S3 challenge repository
5. Register the challenge via the Management API
6. Activate the challenge when ready for participants

## Assessment Engine Capabilities

### Dynamic Assessment Process

The assessment engine uses a sophisticated process to evaluate solutions:

1. Participant requests assessment via API Gateway
2. Request flows through SNS to SQS for buffering/error handling
3. Assessment Lambda polls the queue for new requests
4. Lambda loads challenge configuration from DynamoDB
5. Lambda dynamically loads check functions from S3
6. Lambda assumes a cross-account role in the team account
7. Check functions are executed against participant resources
8. Results are stored in DynamoDB and published via SNS
9. Participant is notified of assessment results

### Dynamic Code Loading

The system safely loads and executes challenge-specific code:

```javascript
// Simplified example of dynamic code loading
async function loadCheckFunctionsFromS3(s3Location, checkFunctionsFile) {
  const s3 = new AWS.S3();
  const bucketName = s3Location.replace('s3://', '').split('/')[0];
  const keyPrefix = s3Location.replace(`s3://${bucketName}/`, '');
  
  const response = await s3.getObject({
    Bucket: bucketName,
    Key: `${keyPrefix}${checkFunctionsFile}`
  }).promise();
  
  // Convert buffer to string
  const functionCode = response.Body.toString('utf-8');
  
  // Create a module from the code string
  return requireFromString(functionCode);
}
```

This enables:
- Challenge-specific assessment logic
- Independent development of new challenges
- Isolation between challenge assessments
- Versioning of assessment criteria

### Cross-Account Assessment

The Assessment Engine uses cross-account roles to securely access and evaluate resources:

```javascript
async function assumeAssessmentRole(accountId) {
  const roleArn = `arn:aws:iam::${accountId}:role/AssessmentEngineAccessRole`;
  const response = await sts.assumeRole({
    RoleArn: roleArn,
    RoleSessionName: "AssessmentEngineSession",
    ExternalId: "ctf-assessment-engine",
    DurationSeconds: 900  // 15 minutes
  }).promise();
  
  return {
    accessKeyId: response.Credentials.AccessKeyId,
    secretAccessKey: response.Credentials.SecretAccessKey,
    sessionToken: response.Credentials.SessionToken
  };
}
```

Each team account has an `AssessmentEngineAccessRole` with least-privilege permissions that only allow:
- Reading CloudFormation stack details
- Examining specific resource configurations
- No modification capabilities
- Time-limited access (15 minutes)

### Check Function Interface

All check functions follow a standardized interface:

```typescript
interface CheckFunctionResult {
  implemented: boolean;              // Whether the feature is implemented
  details?: Record<string, any>;     // Optional details about the implementation
}

type CheckFunction = (
  participantId: string,             // Unique ID of the participant
  stackName: string,                 // CloudFormation stack name
  credentials?: AWS.Credentials      // AWS credentials for the team account
) => Promise<CheckFunctionResult>;
```

Example check function implementation:

```javascript
async function checkMultiRegionDeployment(participantId, stackName, credentials) {
  // If credentials are provided, use them to create clients
  const session = credentials 
    ? new AWS.Session(credentials)
    : new AWS.Session();
  
  // Check for resources in multiple regions
  const regions = ['us-east-1', 'us-west-2', 'eu-west-1'];
  const deployedInRegions = [];
  
  for (const region of regions) {
    const cf = session.client('cloudformation', { region });
    try {
      const stack = await cf.describeStacks({ StackName: stackName }).promise();
      if (stack.Stacks && stack.Stacks.length > 0) {
        deployedInRegions.push(region);
      }
    } catch (error) {
      // Stack doesn't exist in this region, continue
    }
  }
  
  // Check for global tables
  const dynamodb = session.client('dynamodb');
  let hasGlobalTables = false;
  // Implementation continues...
  
  return {
    implemented: deployedInRegions.length > 1 || hasGlobalTables,
    details: {
      regions: deployedInRegions,
      hasGlobalTables
    }
  };
}
```

### Scoring and Feedback Generation

The Assessment Engine computes scores and generates detailed feedback:

```javascript
function calculateScore(assessmentResults) {
  // Sum points for implemented criteria
  return assessmentResults.reduce((total, result) => 
    total + (result.implemented ? result.points : 0), 0);
}

function generateFeedback(assessmentResults, score, passed) {
  const implemented = [];
  const suggestions = [];
  
  // Process results into feedback categories
  for (const result of assessmentResults) {
    if (result.implemented) {
      implemented.push({
        name: result.name,
        details: result.details
      });
    } else {
      suggestions.push({
        name: result.name,
        points: result.maxPoints,
        suggestion: result.suggestion
      });
    }
  }
  
  // Generate summary message
  let summary;
  if (passed) {
    summary = `Congratulations! You've successfully passed the challenge with a score of ${score}.`;
  } else {
    summary = `You've scored ${score} points, but need more improvements to pass the challenge.`;
  }
  
  return {
    summary,
    implemented,
    suggestions
  };
}
```

This provides:
- Objective scoring based on implementation
- Constructive feedback for incomplete solutions
- Detailed explanations of each assessment criterion
- Learning opportunities even when failing challenges

### Security Measures

The Assessment Engine includes several security measures:

1. **Code Validation**
   - Check functions are scanned for malicious patterns
   - Forbidden APIs and operations are blocked
   - Resource limits prevent abuse

2. **Execution Environment**
   - Isolated Lambda environment for each assessment
   - No network access except to required AWS services
   - Read-only file system
   - Environment variables withheld from check functions

3. **Least Privilege Access**
   - Time-limited credentials
   - Minimal permissions for assessment
   - Session policies restrict allowed actions
   - Comprehensive audit logging

4. **Error Containment**
   - Failures in one check function don't affect others
   - Timeouts prevent infinite loops
   - Memory limits prevent resource exhaustion
   - DLQ captures failed assessments for analysis

## Integration Between Components

The Challenge Management and Assessment Engine components work together seamlessly:

1. Challenge developers create and register challenges via the Management API
2. Challenges are stored in S3 and registered in DynamoDB
3. Participants access challenges through the CTF platform
4. When ready for assessment, the Assessment Engine:
   - Retrieves the challenge definition
   - Loads the appropriate check functions
   - Evaluates the solution
   - Provides scoring and feedback

This separation of concerns allows:
- Multiple challenge developers to work independently
- Standardized assessment across all challenges
- Consistent participant experience
- Scalability as more challenges are added

The dynamic nature of the system means new challenges can be added at any time without modifying or redeploying the core Assessment Engine, making it a flexible platform for running AWS Reliability CTF events.
