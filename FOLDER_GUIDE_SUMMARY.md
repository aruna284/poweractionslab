# Folder Structure - Complete Guide

## 🎯 Quick Answer to Your Question

**Where are these folders created?**

| Folder | Location | Created During | Persists |
|--------|----------|---|---|
| `solutions/` | **Git Repository** (Persistent) | Export workflow | ✅ YES |
| `out/ship/` | **GitHub Actions Runner** | Workflow execution | ❌ NO |
| `out/solutions/` | **GitHub Actions Runner** | Workflow execution | ❌ NO |
| `out/release/` | **GitHub Actions Runner** | Workflow execution | ❌ NO |
| `out/exported/` | **GitHub Actions Runner** | Export workflow | ❌ NO |

---

## 🔴 Most Important Concept

**The `out/` folders (out/ship/, out/solutions/, out/release/, out/exported/) are created on the GitHub Actions runner machine (GitHub's cloud servers), NOT in your repository.**

These temporary folders exist only while the workflow is running. Once the job completes, the folder and all its contents are deleted. You can't see them, access them, or recover them - they're automatically cleaned up.

---

## 📍 Visual Locations

### Permanent Location (Your Repository)
```
GitHub Repository: aruna284/poweractionslab
├── .github/
│   └── workflows/
│       ├── Export Solution from Dev.yml
│       └── release-solution-to-prod.yml
│
├── solutions/  ← YOU CAN SEE THIS ON GITHUB
│   └── Model_driven_soln_for_github/
│       ├── CanvasApps/
│       ├── Flows/
│       ├── Other/
│       │   └── Customizations.xml
│       └── ... (other components)
│
├── WORKFLOWS.md
├── FOLDER_STRUCTURE.md
├── FOLDER_LOCATIONS.txt
└── ... (other files)
```

**Access:** On GitHub.com → Code tab → Click folders and files

---

### Temporary Locations (GitHub Actions Runner - Ephemeral)
```
GitHub Actions Runner (Windows Server)
[Only exists during workflow execution]

D:\\ (or C:\\)
├── actions-runner/
│   ├── _work/
│   │   ├── poweractionslab/  ← Repository checked out here
│   │   │   ├── .github/
│   │   │   ├── solutions/
│   │   │   └── ... (repo files)
│   │   │
│   │   └── out/  ← TEMPORARY (created by workflow)
│   │       ├── ship/         ← Not used in release workflow
│   │       ├── solutions/    ← Where managed ZIP is created
│   │       ├── release/      ← Where ZIP is downloaded for deployment
│   │       └── exported/     ← Where exports are stored (Export workflow)
│   │
│   └── [After job completes - all deleted]
```

**Access:** Not visible to you - exists only during workflow

---

## 🔄 How Folders Are Used in Each Workflow

### Export Workflow: `Export Solution from Dev.yml`

```
Step 1: Authenticate to DEV Environment
       └─→ Service Principal logs in

Step 2: Export Unmanaged Solution
       Input: Power Platform DEV
       Output: out/exported/{solution_name}.zip
       Location: GitHub Actions Runner

Step 3: Export Managed Solution (Backup)
       Input: Power Platform DEV  
       Output: out/exported/{solution_name}_managed.zip
       Location: GitHub Actions Runner

Step 4: Unpack Solution
       Input: out/exported/{solution_name}.zip
       Output: out/solutions/{solution_name}/
       Location: GitHub Actions Runner
       Contains: Individual solution components

Step 5: Branch & Commit
       Input: out/solutions/{solution_name}/
       Output: solutions/{solution_name}/ (in Git)
       Location: GitHub Repository ← PERSISTED
       Creates: Pull Request for review

Step 6: Cleanup
       Delete: out/exported/ (temporary)
       Delete: out/solutions/ (temporary)
       Keep: solutions/ (in Git repository)
```

**Final Result:** Solution files in `solutions/` directory are committed to GitHub and persist.

---

### Release Workflow: `release-solution-to-prod.yml`

#### Job 1: convert-to-managed

```
Step 1: Checkout Repository
       Source: GitHub → solutions/{solution_name}/
       Destination: GitHub Actions Runner workspace
       Purpose: Get the unpacked solution files

Step 2: Pack Solution
       Input: solutions/{solution_name}/ (unpacked)
       Output: out/solutions/{solution_name}_managed.zip
       Location: GitHub Actions Runner
       Action: pack-solution

Step 3: Upload to Artifacts
       Input: out/solutions/{solution_name}_managed.zip
       Output: managedSolutions artifact
       Location: GitHub Cloud (Artifacts storage)
       Purpose: Pass to next jobs

Step 4: Cleanup
       Delete: out/solutions/
       (Will be created fresh in next job if needed)
```

---

#### Job 2: release-to-uat

```
Step 1: Checkout Repository
       Source: GitHub → (minimal checkout)
       Destination: GitHub Actions Runner workspace

Step 2: Download from Artifacts
       Input: managedSolutions artifact
       Output: out/release/{solution_name}_managed.zip
       Location: GitHub Actions Runner
       Action: download-artifact

Step 3: Deploy to UAT
       Input: out/release/{solution_name}_managed.zip
       Destination: UAT Power Platform Environment
       Action: import-solution
       Authenticates using Service Principal

Step 4: Cleanup
       Delete: out/release/
```

---

#### Job 3: release-to-prod

```
Step 1: Checkout Repository
       Source: GitHub → (minimal checkout)
       Destination: GitHub Actions Runner workspace

Step 2: Download from Artifacts
       Input: managedSolutions artifact
       Output: out/release/{solution_name}_managed.zip
       Location: GitHub Actions Runner
       Action: download-artifact

Step 3: Deploy to PROD
       Input: out/release/{solution_name}_managed.zip
       Destination: PROD Power Platform Environment
       Action: import-solution
       Authenticates using Service Principal

Step 4: Cleanup
       Delete: out/release/
```

---

## 📊 Folder Lifecycle

### `out/solutions/` Folder

```
Job 1 Timeline
├── 00:00 - Workflow starts
│
├── 00:30 - Repository checked out
│          solutions/ folder is now available
│
├── 01:00 - pack-solution runs
│          out/solutions/ created
│          Model_driven_soln_for_github_managed.zip written
│          Size: ~10-50 MB
│
├── 01:20 - upload-artifact runs
│          ZIP copied to GitHub Artifacts
│
├── 01:40 - Artifacts uploaded successfully
│
├── 01:50 - Job cleanup starts
│          out/solutions/ folder DELETED
│          (temporary files cleaned up)
│
└── 02:00 - Job complete
           out/solutions/ NO LONGER EXISTS
           (only exists in GitHub Artifacts now)

Job 2 Timeline (runs AFTER Job 1)
├── 00:00 - Job 2 starts
│          out/solutions/ does NOT exist (different runner)
│
├── 00:30 - Repository checked out
│
├── 01:00 - download-artifact runs
│          out/release/ created
│          ZIP downloaded from artifacts into out/release/
│
├── 02:30 - import-solution completes
│          ZIP successfully deployed to UAT
│
├── 02:45 - Job cleanup
│          out/release/ deleted
│
└── 03:00 - Job complete
           out/release/ NO LONGER EXISTS
```

**Key Point:** Each job runs on a different GitHub Actions runner machine. They don't share the filesystem. That's why we use GitHub Artifacts to transfer files between jobs.

---

## 🎯 What This Means For You

### ✅ You Can Access/See:
- ✅ `solutions/` folder in your GitHub repository
- ✅ Pull requests created by Export workflow
- ✅ Workflow execution logs in Actions tab
- ✅ Artifacts in workflow run details (GitHub UI)

### ❌ You CANNOT See:
- ❌ `out/` folders (they don't persist)
- ❌ Temporary files on GitHub Actions runner
- ❌ Folder contents after workflow completes
- ❌ Intermediate files that are auto-cleaned

### ✅ Where to Look for Information:
- **Exported files:** GitHub → Pull Requests → Review PR
- **Solution files:** GitHub → Code → solutions/ folder
- **Workflow logs:** GitHub → Actions → workflow run → Logs
- **Artifacts:** GitHub → Actions → workflow run → Artifacts

---

## 🤔 Why This Design?

### Why use temporary folders on the runner?
1. **Efficiency** - Don't bloat repository with large ZIPs
2. **Performance** - Temporary files are fast to create/delete
3. **Clean separation** - Don't mix temporary and permanent data
4. **Scalability** - Runners can be discarded and recreated

### Why use GitHub Artifacts for inter-job communication?
1. **Secure** - Files encrypted in transit
2. **Reliable** - Automatic retry on failure
3. **Convenient** - Actions handle all the details
4. **Recoverable** - Can download from GitHub UI

### Why keep `solutions/` in Git?
1. **Version control** - Track all changes to solution
2. **Audit trail** - See who changed what when
3. **Collaboration** - Pull requests for code review
4. **Backup** - Always have a copy in GitHub
5. **Reference** - Can clone and work locally

---

## 📋 Summary Table

| Folder | Type | Location | When Exists | Access | Purpose |
|--------|------|----------|-------------|--------|---------|
| `solutions/{name}/` | Permanent | Git Repo | Always | GitHub UI, CLI | Version control |
| `out/ship/` | Temporary | Runner | During workflow | Logs only | Staging (unused) |
| `out/solutions/` | Temporary | Runner | Job 1 only | Logs only | Packing staging |
| `out/release/` | Temporary | Runner | Jobs 2-3 only | Logs only | Deploy staging |
| `out/exported/` | Temporary | Runner | Export workflow | Logs only | Export staging |
| `managedSolutions` | Cloud | GitHub Artifacts | ~90 days | GitHub UI, Actions | Inter-job transfer |

---

## ✅ Key Takeaways

1. **Most folders are temporary and ephemeral** - They exist during workflow, then disappear
2. **Only `solutions/` persists** - Everything else is temporary
3. **GitHub Artifacts enable job communication** - Files automatically transferred between jobs
4. **You can only see `solutions/` folder** - Other `out/` folders only exist during execution
5. **Workflow handles everything automatically** - No manual folder creation needed

---

## 🆘 If You Need to Debug

**To see what happened during workflow:**
1. Go to GitHub → Actions → workflow run
2. Click the job name (Job 1, Job 2, etc.)
3. Scroll through the logs
4. Look for errors or messages about folders
5. Check Artifacts section for outputs

**To see solution files:**
1. Go to GitHub → Code tab
2. Navigate to `solutions/` folder
3. Browse the directory structure
4. Click files to view contents

**To understand the flow:**
1. Read WORKFLOWS.md (comprehensive guide)
2. Read FOLDER_STRUCTURE.md (detailed reference)
3. Read FOLDER_LOCATIONS.txt (quick reference)
4. Check workflow files for exact paths used

---

## 📞 Common Questions

**Q: Can I create the `out/` folders in my repository?**  
A: You could, but you shouldn't. Keep them temporary. The workflow creates them as needed.

**Q: What if I need to keep the temporary files?**  
A: Add a `upload-artifact` step to save them to GitHub Artifacts for later download.

**Q: Where are my solution files after export?**  
A: In the `solutions/` folder in your repository (committed to GitHub).

**Q: Where does the managed ZIP go after packing?**  
A: To GitHub Artifacts, then downloaded to `out/release/` before deployment.

**Q: Can I modify the folder paths?**  
A: Yes, but keep defaults - they follow best practices.

**Q: Do temporary folders take up space in my repository?**  
A: No. GitHub only stores what you explicitly commit or upload as artifacts.

---

See **FOLDER_STRUCTURE.md** for more detailed information with data flow diagrams.
