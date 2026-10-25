# Power Platform ALM Workflows

This repository contains a complete **Application Lifecycle Management (ALM)** pipeline for Power Platform solutions using GitHub Actions. The pipeline automates solution export, packing, and deployment across DEV → UAT → PROD environments.

---

## 📋 Workflow Overview

### Two-Workflow ALM Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│ WORKFLOW 1: Export Solution from DEV                            │
│ (First Stage - Pull from Power Platform)                        │
├─────────────────────────────────────────────────────────────────┤
│  • Exports solution from DEV environment                        │
│  • Creates unmanaged (versioned) and managed (backup) ZIPs      │
│  • Unpacks solution for Git version control                     │
│  • Generates pull request for code review                       │
│  • Duration: 3-5 minutes                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                        [Merge PR to main]
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ WORKFLOW 2: Release Solution to UAT/PROD                        │
│ (Final Stage - Push to Environments)                            │
├─────────────────────────────────────────────────────────────────┤
│  Job 1: convert-to-managed                                      │
│    • Packs unpacked solution → managed ZIP                      │
│    • Stores ZIP in GitHub artifacts (1-2 min)                   │
│                                                                  │
│  Job 2: release-to-uat (conditional, depends on Job 1)          │
│    • Downloads managed ZIP from artifacts                       │
│    • Deploys to UAT environment (2-3 min)                       │
│    • Publishes all customizations                               │
│                                                                  │
│  Job 3: release-to-prod (conditional, depends on Job 2)         │
│    • Downloads managed ZIP from artifacts                       │
│    • Deploys to PROD environment (2-3 min)                      │
│    • Publishes all customizations                               │
│    • SAFETY: Only runs after UAT succeeds                       │
│                                                                  │
│  Total Duration: 5-8 minutes (depends on which jobs enabled)    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Workflow 1: Export Solution from DEV

**File:** `.github/workflows/Export Solution from Dev.yml`

### Purpose
This is the **first stage** of the ALM pipeline. It exports solutions from your DEV Power Platform environment, unpacks them for version control, and creates a pull request for review before promoting to UAT/PROD.

### What It Does

1. **Authenticates** to DEV environment using Service Principal credentials
2. **Exports unmanaged solution** → For Git version control and development
3. **Exports managed solution** → Backup copy for deployment
4. **Unpacks solution** → Individual files for easy Git tracking
5. **Creates Git branch** → Contains all solution components
6. **Generates pull request** → For team review before merging

### Inputs

| Input | Required | Default | Purpose |
|-------|----------|---------|---------|
| `solution_name` | ✅ Yes | `Model_driven_soln_for_github` | Name of solution to export from DEV (must exist) |
| `solution_exported_folder` | ❌ No | `out/exported/` | [INTERNAL] Temporary staging folder |
| `solution_folder` | ❌ No | `out/solutions/` | [INTERNAL] Unpacking staging folder |
| `solution_target_folder` | ❌ No | `solutions/` | [INTERNAL] Final Git commit folder |

### Environment Variables (Edit These)

```yaml
ENVIRONMENT_URL: 'https://org7f00f137.crm3.dynamics.com/'   # Your DEV environment URL
CLIENT_ID: 'd2fa770c-78cb-4bfe-a63c-eae5961a5632'          # Service Principal App ID
TENANT_ID: '3f1ffa42-f017-454c-8887-88a2dbf95dd5'          # Azure Tenant ID
```

### GitHub Secrets Required

| Secret | Purpose | Example |
|--------|---------|---------|
| `PowerPlatformSPN` | Service Principal client secret (encrypted) | `[client-secret-value]` |

### Outputs

- ✅ New Git branch: `{solution_name}-{timestamp}`
- ✅ Pull request with all solution changes
- ✅ Unpacked solution files in: `solutions/{solution_name}/`
- ✅ Directory structure includes all customizations

### Typical Usage

```bash
# Trigger the workflow via GitHub UI or CLI
gh workflow run "Export Solution from Dev.yml" \
  -f solution_name="MyPowerPlatformSolution"

# Result:
# 1. Workflow exports from DEV
# 2. Creates new branch with unpacked files
# 3. Opens PR for review
# 4. Review and merge PR to main
# 5. Solution files ready for deployment workflow
```

### Next Steps

After export completes:
1. Go to **Pull Requests** in GitHub
2. Review the changes in `solutions/` directory
3. Approve and merge the PR
4. Solution is ready for promotion to UAT/PROD

---

## 🚀 Workflow 2: Release Solution to UAT and PROD

**File:** `.github/workflows/release-solution-to-prod.yml`

### Purpose
This is the **final stage** of the ALM pipeline. It packs solutions and deploys them to UAT and/or Production environments with optional selective deployment. It includes built-in safeguards to test in UAT before PROD.

### What It Does

#### Job 1: convert-to-managed (Always Runs)
- Packs unpacked solution files into managed ZIP format
- Uploads ZIP to GitHub artifacts for downstream jobs
- Duration: ~1-2 minutes

#### Job 2: release-to-uat (Conditional - runs if `DEPLOY_TO_UAT=true`)
- Downloads managed ZIP from artifacts
- Authenticates with Service Principal
- Imports solution to UAT environment
- Publishes all customizations in UAT
- Duration: ~2-3 minutes

#### Job 3: release-to-prod (Conditional - runs if `DEPLOY_TO_PROD=true` AND release-to-uat succeeds)
- Downloads managed ZIP from artifacts
- Authenticates with Service Principal
- Imports solution to PROD environment
- Publishes all customizations in PROD
- Duration: ~2-3 minutes

### Inputs

| Input | Required | Default | Purpose |
|-------|----------|---------|---------|
| `solution_name` | ✅ Yes | `<SolutionName>` | Solution to deploy (must exist in solutions/) |
| `DEPLOY_TO_UAT` | ❌ No | `true` | Enable UAT deployment |
| `DEPLOY_TO_PROD` | ❌ No | `false` | Enable PROD deployment |
| `UAT_ENVIRONMENT_URL` | ✅ Yes | `https://orgcf4bd8b1.crm3.dynamics.com/` | UAT environment URL |
| `PRODUCTION_ENVIRONMENT_URL` | ✅ Yes | `https://org14021c04.crm3.dynamics.com/` | PROD environment URL |
| `CLIENT_ID` | ✅ Yes | `d2fa770c-78cb-4bfe-a63c-eae5961a5632` | Service Principal App ID |
| `TENANT_ID` | ✅ Yes | `3f1ffa42-f017-454c-8887-88a2dbf95dd5` | Azure Tenant ID |

### GitHub Secrets Required

| Secret | Purpose | Example |
|--------|---------|---------|
| `PowerPlatformSPN` | Service Principal client secret (encrypted) | `[client-secret-value]` |

### Deployment Modes

#### Mode 1: Test in UAT Only
```bash
gh workflow run release-solution-to-prod.yml \
  -f solution_name="MyApp" \
  -f DEPLOY_TO_UAT=true \
  -f DEPLOY_TO_PROD=false
```
- Jobs 1 & 2 run ✅
- Job 3 skipped (PROD protection)
- Test your changes in UAT

#### Mode 2: Deploy to Both UAT and PROD
```bash
gh workflow run release-solution-to-prod.yml \
  -f solution_name="MyApp" \
  -f DEPLOY_TO_UAT=true \
  -f DEPLOY_TO_PROD=true
```
- All 3 jobs run ✅
- UAT deployment first
- PROD only proceeds after UAT succeeds
- Full deployment pipeline

#### Mode 3: Skip to PROD (Not Recommended)
```bash
gh workflow run release-solution-to-prod.yml \
  -f solution_name="MyApp" \
  -f DEPLOY_TO_UAT=false \
  -f DEPLOY_TO_PROD=true
```
- Jobs 1 & 3 run
- Job 2 (UAT) skipped
- ⚠️ **WARNING**: Bypasses UAT testing

### Outputs

- ✅ `managedSolutions` artifact (ZIP file)
- ✅ Managed solution deployed to UAT (if enabled)
- ✅ Managed solution deployed to PROD (if enabled)
- ✅ All customizations published in target environments

---

## 🔄 Complete ALM Workflow Example

### Scenario: Deploy a new Power Platform solution to Production

**Step 1: Export from DEV (5 minutes)**
```bash
# Run the export workflow
gh workflow run "Export Solution from Dev.yml" \
  -f solution_name="CustomerPortal"

# Monitor the run
gh run list --workflow="Export Solution from Dev.yml" --limit 1
```
- Solution exports from DEV environment
- Branch created with unpacked files
- Pull request opens for review

**Step 2: Code Review & Merge (Variable)**
```bash
# Review the PR changes
gh pr view 1

# Approve and merge
gh pr merge 1 --merge
```
- Team reviews solution changes in `solutions/CustomerPortal/`
- Approves changes
- Merges PR to main branch

**Step 3: Test in UAT (7-8 minutes)**
```bash
# Deploy to UAT for testing
gh workflow run release-solution-to-prod.yml \
  -f solution_name="CustomerPortal" \
  -f DEPLOY_TO_UAT=true \
  -f DEPLOY_TO_PROD=false

# Monitor UAT deployment
gh run list --workflow="release-solution-to-prod.yml" --limit 1
```
- Solution packs to managed format
- Deploys to UAT environment
- Test team validates in UAT

**Step 4: Deploy to PROD (5-8 minutes)**
```bash
# After successful UAT testing, deploy to PROD
gh workflow run release-solution-to-prod.yml \
  -f solution_name="CustomerPortal" \
  -f DEPLOY_TO_UAT=true \
  -f DEPLOY_TO_PROD=true

# Monitor the full pipeline
gh run view <run-id> --log
```
- Solution deploys to PROD
- Workflow only proceeds to PROD after UAT succeeds
- Production live with new version

---

## ⚙️ Configuration Reference

### Service Principal Setup

Both workflows use a **Service Principal** for authentication. This requires:

1. **Azure AD Application Registration**
   - App ID (CLIENT_ID)
   - Client Secret (stored in GitHub Secret: `PowerPlatformSPN`)
   - Tenant ID (TENANT_ID)

2. **Power Platform Environment Registration**
   - Service Principal must be added to DEV, UAT, and PROD environments
   - Must have appropriate permissions (e.g., System Administrator)

3. **GitHub Secret Configuration**
   - Go to: Settings → Secrets and variables → Actions
   - Add secret: `PowerPlatformSPN` = Client Secret value

### Environment URLs

Update these URLs to match your environments:

```yaml
# Export Solution from Dev.yml
ENVIRONMENT_URL: 'https://org7f00f137.crm3.dynamics.com/'  # Your DEV URL

# release-solution-to-prod.yml
UAT_ENVIRONMENT_URL: 'https://orgcf4bd8b1.crm3.dynamics.com/'
PRODUCTION_ENVIRONMENT_URL: 'https://org14021c04.crm3.dynamics.com/'
```

Find your environment URL in Power Platform Admin Center:
1. Go to Environments
2. Select your environment
3. Copy the URL from the details panel

---

## 📊 Workflow Artifacts

### Managed Solutions Artifact
- **Name:** `managedSolutions`
- **Format:** ZIP file
- **Contains:** Managed solution ready for deployment
- **Passed between jobs** via GitHub Artifacts API
- **Retention:** Default (GitHub setting)

### Git Branches
- **Created by:** Export Solution from Dev workflow
- **Name format:** `{solution_name}-{timestamp}`
- **Contains:** Unpacked solution files
- **Status:** Temporary (used for PR, can delete after merge)

### Pull Requests
- **Created by:** Export Solution from Dev workflow
- **Title:** Branch name with changes
- **Body:** Lists all modified files
- **Purpose:** Code review before promotion

---

## 🔒 Security Best Practices

1. **Service Principal Credentials**
   - Client Secret stored only in GitHub Secrets
   - Never hardcode or commit secrets
   - Rotate periodically

2. **PROD Protection**
   - PROD deployment defaults to `false` (disabled)
   - Requires explicit `DEPLOY_TO_PROD=true` to enable
   - Only proceeds after UAT succeeds

3. **Least Privilege**
   - Grant Service Principal minimum required permissions
   - Don't use global admin role
   - Create dedicated SP for ALM pipeline

4. **Audit Trail**
   - All deployments logged in GitHub Actions
   - Pull requests provide change history
   - Dynamics 365 audit logs capture environment changes

---

## 🛠️ Troubleshooting

### Common Issues

#### "Solution not found in DEV"
- **Cause:** Wrong solution name or solution doesn't exist
- **Fix:** Verify solution name in Power Platform admin center

#### "Service principal does not have permission"
- **Cause:** SP not added to environment or insufficient permissions
- **Fix:** Add SP to environment with System Administrator role

#### "PAC is not installed"
- **Cause:** Power Platform CLI installation failed
- **Fix:** Workflow will retry; check Actions logs for details

#### "Solution file not found"
- **Cause:** Export Solution workflow not run or PR not merged
- **Fix:** Run Export workflow, merge PR to main, then retry Release

### Viewing Logs

```bash
# List all workflow runs
gh run list --workflow="release-solution-to-prod.yml"

# View specific run details
gh run view <run-id>

# View job logs
gh run view <run-id> --log

# View specific job details
gh run view --job=<job-id>
```

---

## 📝 Files Modified/Created

```
.github/workflows/
├── Export Solution from Dev.yml         [STAGE 1: Export & Version Control]
└── release-solution-to-prod.yml         [STAGE 2: Pack & Deploy]

solutions/
└── {solution_name}/                     [Unpacked solution files]
    ├── Other/
    │   └── Customizations.xml
    ├── CanvasApps/
    ├── Flows/
    ├── PluginAssemblies/
    └── ... (other components)
```

---

## ✅ Validation Checklist

Before using these workflows, ensure:

- [ ] Service Principal created in Azure AD
- [ ] Service Principal added to DEV, UAT, PROD environments
- [ ] GitHub Secret `PowerPlatformSPN` configured
- [ ] Environment URLs updated in workflow files
- [ ] Solution exists in DEV environment
- [ ] Repository has appropriate permissions
- [ ] Team members have GitHub Actions access

---

## 📚 Additional Resources

- [Microsoft Power Platform Actions](https://github.com/microsoft/powerplatform-actions)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Power Platform ALM Best Practices](https://docs.microsoft.com/en-us/power-platform/alm/)
- [Service Principal in Power Platform](https://docs.microsoft.com/en-us/power-platform/admin/system-application-users)

---

## 📞 Support

For issues or questions:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review workflow logs in GitHub Actions
3. Verify Service Principal configuration
4. Consult Microsoft Power Platform documentation

---

**Last Updated:** October 25, 2026  
**Workflow Status:** ✅ Fully Functional and Tested  
**Latest Validation:** Run #9 - All jobs successful (6m 25s)
