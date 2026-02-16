# 🧪 Auto-Rebase Testing Guide

Complete guide for building, testing, and validating the auto-rebase feature with Compare API-based behind detection, conflict detection, and optional atlantis comment checking.

## ⚠️ Important: How Behind Detection Works

**Naysayer uses GitLab Compare API (not MR fields) to determine if rebase is needed:**

### Why MR Fields Are Unreliable
GitLab MR fields like `behind_commits_count`, `diverged_commits_count`, `merge_status`, and `detailed_merge_status`:
- Can be `null` or `0` even when MR is actually behind
- May be stale or not updated after target branch changes
- Can be blocked/incorrect when approval rules are configured
- Don't consistently reflect the true branch comparison state

### The Authoritative Method: Compare API

**Same-project MRs** (source and target in the same project):
```bash
curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_BASE_URL/api/v4/projects/$PROJECT_ID/repository/compare?from=$SOURCE_BRANCH&to=$TARGET_BRANCH" \
  | jq '{behind_count: (.commits | length), behind_commits: [.commits[].id]}'
```

**Fork MRs** (GitLab REST API cannot compare across projects by branch; use SHAs in the **upstream** project):
```bash
# 1) Get fork branch SHA from the MR (source branch HEAD)
FORK_SHA=$(curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_BASE_URL/api/v4/projects/$UPSTREAM_PROJECT_ID/merge_requests/$MR_IID" \
  | jq -r '.sha')

# 2) Get target branch (e.g. main) SHA in upstream project
MAIN_SHA=$(curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_BASE_URL/api/v4/projects/$UPSTREAM_PROJECT_ID/repository/branches/$TARGET_BRANCH" \
  | jq -r '.commit.id')

# 3) Compare SHAs in the upstream project
curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_BASE_URL/api/v4/projects/$UPSTREAM_PROJECT_ID/repository/compare?from=$FORK_SHA&to=$MAIN_SHA" \
  | jq '{behind_count: (.commits | length), behind_commits: [.commits[].id]}'
```

**How it works:**
- Same-project: `from=source_branch`, `to=target_branch` in one project.
- Fork MRs: GitLab does **not** support cross-project compare by branch (e.g. `from=project_id:branch` returns 404). Naysayer uses **SHA-based compare** in the upstream (target) project: MR `.sha` = source branch HEAD, then compare `from=<MR.sha>&to=<target_branch_sha>`.
- `commits` = commits in target that source doesn’t have; `behind_count = commits.length`.
- If `behind_count > 0` → Rebase needed; if `behind_count == 0` → Up-to-date, skip rebase.

**Direction:** `from=source` (or source SHA), `to=target` (or target SHA).

**Fork MR support:** Naysayer detects fork MRs (`source_project_id != target_project_id`), fetches MR `.sha` and target branch SHA, and calls compare in the upstream project.

## 📋 Overview

This guide covers end-to-end testing of the auto-rebase logic that:
- **Uses GitLab Compare API** to authoritatively determine if MR is behind (not unreliable MR fields)
- Checks pipeline status (success/failed)
- Validates all pipeline jobs succeeded
- Verifies rebase completed successfully
- Optionally checks latest atlantis-bot comments for plan failures (when enabled)
- Distinguishes between state lock failures (allow rebase) and actual plan failures (skip rebase)

## 🏗️ Building Naysayer

### Prerequisites

- Go 1.21 or later
- Git
- GitLab token with appropriate permissions
- Access to target repository for testing

### Step 1: Clone and Setup

```bash
# Clone the repository
git clone <repository-url>
cd naysayer

# Install dependencies
go mod tidy

# Verify Go version
go version
```

### Step 2: Build Binary

```bash
# Build for local testing
go build -o naysayer cmd/main.go

# Or build with optimizations
go build -ldflags="-s -w" -o naysayer cmd/main.go

# Verify build
./naysayer --help
```

### Step 3: Build Container Image (Optional)

```bash
# Build Docker image
docker build -t naysayer:test .

# Or use Makefile
make build-image
```

## 🔧 Configuration Setup

### Step 1: Configure Environment Variables

Create a `.env` file or export variables:

```bash
# Required GitLab configuration
export GITLAB_TOKEN="glpat-your-token"
export GITLAB_BASE_URL="https://gitlab.cee.redhat.com"

# Optional: Repository-specific token (supports both new and legacy env var names)
export AUTO_REBASE_REPOSITORY_TOKEN="glpat-repository-token"
# Or use legacy name for backward compatibility:
export GITLAB_TOKEN_FIVETRAN="glpat-repository-token"

# Logging
export LOG_LEVEL="info"  # or "debug" for verbose output
```

### Step 2: Verify Configuration

```bash
# Test configuration loading
go run cmd/main.go --help

# Check if configuration is valid
# (Application will fail to start if config is invalid)
```

## 🧪 Unit Testing

### Step 1: Run All Tests

```bash
# Run all tests
go test ./...

# Run with verbose output
go test -v ./...

# Run with coverage
go test -cover ./...

# Run specific package tests
go test ./internal/webhook/...
go test ./internal/gitlab/...
```

### Step 2: Test Auto-Rebase Logic

```bash
# Test filterEligibleMRs function
go test -v ./internal/webhook/... -run TestAutoRebaseHandler_FilterEligibleMRs

# Test with coverage
go test -cover ./internal/webhook/... -run TestAutoRebaseHandler
```

### Step 3: Test GitLab Client Methods

```bash
# Test new pipeline job methods
go test -v ./internal/gitlab/... -run TestClient

# Test atlantis comment finding
go test -v ./internal/gitlab/... -run TestFindLatestAtlantisComment
```

## 🔍 Integration Testing

### Step 1: Test Pipeline Job Checking

Use the provided test script to verify pipeline job checking:

```bash
# Set environment variables
export AUTO_REBASE_REPOSITORY_TOKEN="glpat-your-token"
export PROJECT_ID="123"
export MR_IID="456"

# Run the test script
./test_pipeline_failure_reason.sh
```

Or use the Go test program:

```bash
# Run with environment variable
export AUTO_REBASE_REPOSITORY_TOKEN="glpat-your-token"
go run cmd/test_pipeline_jobs.go \
  -project-id 123 \
  -mr-iid 456 \
  -gitlab-url "https://gitlab.cee.redhat.com"

# Show full trace
go run cmd/test_pipeline_jobs.go \
  -project-id 123 \
  -mr-iid 456 \
  -show-trace

# Show more trace lines
go run cmd/test_pipeline_jobs.go \
  -project-id 123 \
  -mr-iid 456 \
  -trace-lines 100
```

### Step 2: Test Atlantis Comment Detection

Create a test MR with atlantis comments and verify detection:

```bash
# Test script checks for atlantis comments
export PROJECT_ID="123"
export MR_IID="456"
./test_pipeline_failure_reason.sh
```

Expected output for state lock:
```
✅ RESULT: FAILED DUE TO STATE LOCK
🔒 This MR is GOOD TO REBASE
```

Expected output for plan failure:
```
❌ RESULT: FAILED DUE TO OTHER REASON
⚠️  This MR should NOT be rebased
```

## 🚀 End-to-End Testing

### Step 1: Start Naysayer Locally

```bash
# Set environment variables
export AUTO_REBASE_REPOSITORY_TOKEN="glpat-your-token"
export GITLAB_BASE_URL="https://gitlab.cee.redhat.com"
export LOG_LEVEL="debug"

# Start server
go run cmd/main.go

# Or use built binary
./naysayer
```

Server should start on port 3000 (or configured port).

### Step 2: Test Webhook Endpoint

#### Test 1: Push to Main Branch (Success Case)

```bash
# Create test payload for push to main
cat > /tmp/push_payload.json <<EOF
{
  "object_kind": "push",
  "ref": "refs/heads/main",
  "project": {
    "id": 123
  }
}
EOF

# Send webhook
curl -X POST http://localhost:3000/auto-rebase \
  -H "Content-Type: application/json" \
  -d @/tmp/push_payload.json
```

Expected response:
```json
{
  "webhook_response": "processed",
  "status": "completed",
  "project_id": 123,
  "branch": "main",
  "total_mrs": 5,
  "eligible_mrs": 3,
  "successful": 3,
  "failed": 0,
  "skipped": 2
}
```

#### Test 2: MR with Successful Pipeline

1. Create an MR with:
   - Pipeline status: `success`
   - (No need to check jobs or atlantis comment - pipeline success means everything passed)

2. Trigger push to main:
```bash
curl -X POST http://localhost:3000/auto-rebase \
  -H "Content-Type: application/json" \
  -d @/tmp/push_payload.json
```

Expected: MR should be rebased (pipeline success means no failures, proceed directly).

#### Test 3: MR with Failed Pipeline Due to State Lock

1. Create an MR with:
   - Pipeline status: `failed`
   - All jobs succeeded (important: jobs must be successful)
   - Latest atlantis comment contains "Error: Error acquiring the state lock" (exact Terraform state lock error)

2. Trigger push to main:
```bash
curl -X POST http://localhost:3000/auto-rebase \
  -H "Content-Type: application/json" \
  -d @/tmp/push_payload.json
```

Expected: MR should be rebased (state lock is temporary, all jobs succeeded).

#### Test 4: MR with Failed Pipeline Due to Plan Error

1. Create an MR with:
   - Pipeline status: `failed`
   - All jobs succeeded (important: jobs must be successful)
   - Latest atlantis comment contains plan error (does NOT contain "Error: Error acquiring the state lock")

2. Trigger push to main:
```bash
curl -X POST http://localhost:3000/auto-rebase \
  -H "Content-Type: application/json" \
  -d @/tmp/push_payload.json
```

Expected: MR should be skipped (actual plan failure detected in atlantis comment - not a state lock error).

#### Test 5: MR with Successful Pipeline

**Note**: If pipeline status is `success`, it means all jobs succeeded. This scenario is not possible - if jobs failed, pipeline status would be `failed`, not `success`.

### Step 3: Verify Logs

Check application logs for detailed information:

```bash
# If running locally
tail -f naysayer.log

# Or check console output
# Look for log entries like:
# - "Checking pipeline jobs for MR"
# - "Checking atlantis comment for plan failures"
# - "Skipping MR due to atlantis plan failure"
# - "State lock detected, allowing rebase"
```

### Step 4: Verify MR Comments

After rebase, check that comments were added to MRs:

```bash
# Use GitLab API to check comments
curl -H "Authorization: Bearer $AUTO_REBASE_REPOSITORY_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/123/merge_requests/456/notes"
```

Expected comment:
```
🤖 **Automated Rebase**

This merge request has been automatically rebased with the latest changes from the target branch.

_This is an automated action triggered by a push to the main branch._
```

## 🐛 Debugging

### Enable Debug Logging

```bash
export LOG_LEVEL="debug"
go run cmd/main.go
```

### Test Individual Components

#### Test Pipeline Jobs API

```bash
# Direct API test
curl -H "Authorization: Bearer $AUTO_REBASE_REPOSITORY_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/123/pipelines/456/jobs"
```

#### Test Atlantis Comment Finding

```bash
# Direct API test
curl -H "Authorization: Bearer $AUTO_REBASE_REPOSITORY_TOKEN" \
  "https://gitlab.cee.redhat.com/api/v4/projects/123/merge_requests/456/notes" | \
  jq '.[] | select(.author.username | contains("atlantis"))'
```

### Common Issues

#### Issue: "Failed to get pipeline jobs"

**Solution**: Verify GitLab token has `read_api` scope and access to the project.

#### Issue: "No atlantis comments found"

**Solution**: 
- Verify atlantis-bot username pattern matches detection logic
- Check if comments exist in the MR
- Review comment author username/name fields

#### Issue: "State lock not detected"

**Solution**:
- Check atlantis comment body for the exact error message: "Error: Error acquiring the state lock"
- Verify comment is the latest atlantis comment
- Only this specific error message triggers state lock detection - all other errors will skip rebase

## 📊 Test Scenarios Summary

| Scenario | Pipeline Status | Jobs Status | Atlantis Comment | Expected Result |
|----------|----------------|-------------|------------------|-----------------|
| 1 | success | N/A (not checked) | N/A (not checked) | ✅ Rebase |
| 2 | failed | all succeeded | state lock | ✅ Rebase |
| 3 | failed | all succeeded | plan error | ❌ Skip |
| 4 | failed | some failed | N/A | ❌ Skip |
| 5 | running | N/A | N/A | ❌ Skip |
| 6 | pending | N/A | N/A | ❌ Skip |

**Logic Flow:**
- **Pipeline = success**: Rebase directly (no job or atlantis check needed - success means no failures)
- **Pipeline = failed**: Check jobs → If all succeeded → Check atlantis comment → Rebase if state lock, skip if plan error

## ✅ Pre-Deployment Checklist

Before pushing changes to main, verify:

- [ ] All unit tests pass
- [ ] Integration tests pass
- [ ] Test script (`test_pipeline_failure_reason.sh`) works correctly
- [ ] Go test program (`cmd/test_pipeline_jobs.go`) works correctly
- [ ] End-to-end test with real MRs passes
- [ ] Logs show correct decision making
- [ ] MR comments are added correctly
- [ ] No linting errors
- [ ] Code coverage meets requirements
- [ ] Documentation updated

## 🚀 Deployment Steps

### Step 1: Build and Test Locally

```bash
# Build
go build -o naysayer cmd/main.go

# Run tests
go test ./...

# Test locally
./naysayer
```

### Step 2: Create Test Branch

```bash
git checkout -b feature/autorebase-enhancement
git add .
git commit -m "feat: enhance auto-rebase with pipeline jobs and atlantis comment checking"
git push origin feature/fivetran-autorebase-enhancement
```

### Step 3: Create Merge Request

1. Create MR in GitLab
2. Wait for CI/CD pipeline to pass
3. Review code changes
4. Test in staging environment (if available)

### Step 4: Deploy to Production

```bash
# Merge to main
git checkout main
git merge feature/fivetran-autorebase-enhancement
git push origin main

# Or merge via GitLab UI
```

### Step 5: Monitor Deployment

```bash
# Check deployment status
kubectl get pods -n <namespace>

# Check logs
kubectl logs -f deployment/naysayer -n <namespace>

# Monitor webhook endpoint
# Check for successful rebase operations
```

## 📚 Related Documentation

- [Auto-Rebase Rule Documentation](rules/AUTOREBASE_RULE_AND_SETUP.md)
- [Development Guide](../DEVELOPMENT.md)
- [Deployment Guide](../DEPLOYMENT.md)
- [Testing Pipeline Failure Reasons](../TEST_PIPELINE_FAILURE_REASON.md)

## 🔗 Useful Commands

```bash
# Run specific test
go test -v ./internal/webhook/... -run TestName

# Build with race detector
go test -race ./...

# Check code coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Lint code
golangci-lint run

# Format code
go fmt ./...
```

---

**Last Updated**: 2025-01-XX
**Version**: 1.0.0

