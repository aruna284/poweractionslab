# Folder Structure and Artifact Storage

This document explains where all the folders in the ALM workflows are created, how they're used, and what data they contain.

---

## 📁 Folder Types

There are two types of folders in the ALM pipeline:

### 1️⃣ **Permanent Folders** (In Git Repository)
These folders are committed to GitHub and persist in your repository.

### 2️⃣ **Temporary Folders** (In GitHub Actions Runner)
These folders are created during workflow execution on the GitHub Actions runner and are discarded after the job completes. They do NOT persist in the repository.

---

## 🔍 Folder Breakdown

### **Permanent Folders** (Checked Into Git)

#### 📂 `solutions/` - SOURCE FOLDER
- **Location:** Repository root → `solutions/`
- **Purpose:** Stores unpacked solution files for version control
- **Created By:** Export Solution from Dev workflow (via `branch-solution` action)
- **Persists:** ✅ YES (committed to Git)
- **Contains:** 
  - `solutions/{solution_name}/` - Unpacked solution folder
  - `solutions/{solution_name}/Other/Customizations.xml` - Main solution definition
  - Various component folders (forms, flows, plugins, etc.)
- **Example Structure:**
  ```
  solutions/
  └── Model_driven_soln_for_github/
      ├── CanvasApps/
      ├── Flows/
      ├── FormXml/
      ├── Workflows/
      ├── PluginAssemblies/
      ├── Other/
      │   └── Customizations.xml
      └── ... (other components)
  ```
- **Git Status:** All files tracked and committed
- **Access:** Can be viewed on GitHub, cloned locally, modified offline

---

### **Temporary Folders** (GitHub Actions Runner Only)

These folders are created during workflow execution **inside the GitHub Actions runner environment**. They are NOT part of your repository and are automatically cleaned up after each job.

#### 📂 `out/ship/` - SHIPPING FOLDER
- **Location:** GitHub Actions runner → `out/ship/`
- **Created By:** Automatically created when workflow runs
- **Purpose:** Staging area for exported solutions (currently unused in release workflow)
- **Persists:** ❌ NO (temporary, discarded after workflow)
- **Defined In:** `solution_shipping_folder` input parameter
- **Contains:** (Empty in current workflow - reserved for future use)

---

#### 📂 `out/solutions/` - OUTBOUND FOLDER
- **Location:** GitHub Actions runner → `out/solutions/`
- **Created By:** Automatically created when workflow runs
- **Purpose:** Staging folder where the managed solution ZIP is created
- **Persists:** ❌ NO (temporary, discarded after workflow)
- **Defined In:** `solution_outbound_folder` input parameter
- **Used By:** `pack-solution` action creates the managed ZIP here
- **Contains:** 
  - `Model_driven_soln_for_github_managed.zip` (the packed managed solution)
- **Flow:**
  1. Input: Unpacked solution files from `solutions/` directory
  2. Action: `pack-solution` converts to managed format
  3. Output: ZIP file created at `${{ inputs.solution_outbound_folder}}/`
  4. Artifact Upload: ZIP uploaded to GitHub Artifacts
  5. Cleanup: Folder discarded after job completes

---

#### 📂 `out/release` - RELEASE FOLDER
- **Location:** GitHub Actions runner → `out/release`
- **Created By:** Automatically created when workflow runs
- **Purpose:** Staging folder where artifacts are downloaded before deployment
- **Persists:** ❌ NO (temporary, discarded after workflow)
- **Defined In:** `solution_release_folder` input parameter
- **Used By:** `download-artifact` action downloads managed ZIP here
- **Contains:** 
  - `Model_driven_soln_for_github_managed.zip` (downloaded from artifacts)
- **Flow:**
  1. Download: GitHub Artifacts downloaded to this folder
  2. Deploy: `import-solution` action reads ZIP from here
  3. Cleanup: Folder discarded after job completes

---

#### 📂 `out/exported/` - EXPORT FOLDER (Export Workflow Only)
- **Location:** GitHub Actions runner → `out/exported/`
- **Created By:** Export Solution from Dev workflow
- **Purpose:** Temporary storage for exported ZIPs from Power Platform
- **Persists:** ❌ NO (temporary, discarded after workflow)
- **Defined In:** `solution_exported_folder` input parameter (in Export workflow)
- **Contains:**
  - `{solution_name}.zip` - Unmanaged solution export
  - `{solution_name}_managed.zip` - Managed solution export
- **Flow:**
  1. Export: Power Platform `export-solution` action downloads ZIPs
  2. Unpack: `unpack-solution` action unpacks unmanaged ZIP
  3. Branch: `branch-solution` action commits unpacked files
  4. Cleanup: Folder discarded after workflow

---

## 🔄 Data Flow Through Folders

### **Export Workflow: DEV → Repository**

```
[Power Platform DEV Environment]
            ↓
   export-solution action
            ↓
    out/exported/
    ├── Model_driven_soln_for_github.zip (unmanaged)
    └── Model_driven_soln_for_github_managed.zip (managed)
            ↓
    unpack-solution action
            ↓
    out/solutions/
    └── Model_driven_soln_for_github/
            ↓
    branch-solution action
            ↓
    [Git Repository - PERSISTENT]
    └── solutions/
        └── Model_driven_soln_for_github/
                ↓
        (Files committed to GitHub)
```

---

### **Release Workflow: Repository → UAT/PROD**

```
[Git Repository - PERSISTENT]
└── solutions/
    └── Model_driven_soln_for_github/
                ↓
    [Job 1: convert-to-managed]
    pack-solution action reads from solutions/
                ↓
    out/solutions/
    └── Model_driven_soln_for_github_managed.zip
                ↓
    [GitHub Artifacts - Temporary Storage]
    (ZIP stored for inter-job transfer)
                ↓
    [Job 2: release-to-uat]
    download-artifact action
                ↓
    out/release/
    └── Model_driven_soln_for_github_managed.zip
                ↓
    import-solution action
                ↓
    [Power Platform UAT Environment]
                ↓
    [Job 3: release-to-prod]
    download-artifact action
                ↓
    out/release/
    └── Model_driven_soln_for_github_managed.zip
                ↓
    import-solution action
                ↓
    [Power Platform PROD Environment]
```

---

## 🎯 Folder Usage by Action

### **Export Solution from Dev Workflow**

| Action | Input Folder | Output Folder | Creates? |
|--------|-------------|---------------|----------|
| `export-solution` | (Power Platform) | `out/exported/` | ✅ Creates |
| `unpack-solution` | `out/exported/` | `out/solutions/` | ✅ Creates |
| `branch-solution` | `out/solutions/` | `solutions/` (Git) | ✅ Creates |

---

### **Release Solution to PROD Workflow**

#### Job 1: convert-to-managed
| Action | Input Folder | Output Folder | Creates? |
|--------|-------------|---------------|----------|
| Checkout | (Git) | (Repo on runner) | - |
| `pack-solution` | `solutions/{name}/` | `out/solutions/` | ✅ Creates |
| `upload-artifact` | `out/solutions/` | (GitHub Artifacts) | ✅ Stores |

#### Job 2: release-to-uat
| Action | Input Folder | Output Folder | Creates? |
|--------|-------------|---------------|----------|
| Checkout | (Git) | (Repo on runner) | - |
| `download-artifact` | (GitHub Artifacts) | `out/release/` | ✅ Creates |
| `import-solution` | `out/release/` | (UAT Environment) | ✅ Deploys |

#### Job 3: release-to-prod
| Action | Input Folder | Output Folder | Creates? |
|--------|-------------|---------------|----------|
| Checkout | (Git) | (Repo on runner) | - |
| `download-artifact` | (GitHub Artifacts) | `out/release/` | ✅ Creates |
| `import-solution` | `out/release/` | (PROD Environment) | ✅ Deploys |

---

## 💾 GitHub Artifacts Storage

### What are Artifacts?
GitHub Artifacts are a way to store files between jobs in the same workflow. They're stored temporarily on GitHub's servers.

### managedSolutions Artifact

**Purpose:** Pass the managed solution ZIP between jobs

**Created By:** Job 1 (convert-to-managed)
```yaml
- name: Upload managed solution to GitHub Artifacts
  uses: actions/upload-artifact@v4
  with:
    name: managedSolutions
    path: ${{ inputs.solution_outbound_folder}}
```

**Downloaded By:** Job 2 (release-to-uat) and Job 3 (release-to-prod)
```yaml
- name: Download managed solution from artifacts
  uses: actions/download-artifact@v4
  with:
    name: managedSolutions
    path: ${{ inputs.solution_release_folder}}
```

**Storage Location:** GitHub's artifact storage (not your repository)

**Retention:** Default GitHub policy (usually 90 days or configurable)

**Size:** Typically 9-50 MB (depending on solution complexity)

**Example Workflow:**
```
Job 1 creates: Model_driven_soln_for_github_managed.zip
                        ↓
    Uploads to: managedSolutions artifact
                        ↓
Job 2 downloads from: managedSolutions artifact
Job 3 downloads from: managedSolutions artifact
```

---

## 🗂️ Complete Folder Map

### During Export Workflow (GitHub Actions Runner)

```
GitHub Actions Runner
├── [Repository Checked Out]
│   ├── .github/
│   ├── solutions/
│   │   └── Model_driven_soln_for_github/  ← PERMANENT (from Git)
│   └── ...
│
├── out/  ← TEMPORARY (created during workflow)
│   ├── exported/
│   │   ├── Model_driven_soln_for_github.zip
│   │   └── Model_driven_soln_for_github_managed.zip
│   │
│   └── solutions/
│       └── Model_driven_soln_for_github/
│           ├── CanvasApps/
│           ├── Flows/
│           ├── Other/
│           │   └── Customizations.xml
│           └── ...
│
└── [After Push]
    └── GitHub Repository
        └── solutions/
            └── Model_driven_soln_for_github/  ← PERSISTED
                ├── CanvasApps/
                ├── Flows/
                ├── Other/
                │   └── Customizations.xml
                └── ...
```

### During Release Workflow (GitHub Actions Runner)

```
GitHub Actions Runner - Job 1 (convert-to-managed)
├── [Repository Checked Out]
│   └── solutions/
│       └── Model_driven_soln_for_github/  ← PERMANENT
│
├── out/  ← TEMPORARY
│   └── solutions/
│       └── Model_driven_soln_for_github_managed.zip
│
└── GitHub Artifacts
    └── managedSolutions/
        └── Model_driven_soln_for_github_managed.zip

GitHub Actions Runner - Job 2 (release-to-uat)
├── [Repository Checked Out]
│
├── out/  ← TEMPORARY
│   └── release/
│       └── Model_driven_soln_for_github_managed.zip
│
└── GitHub Artifacts (Downloaded)
    └── managedSolutions/
        └── Model_driven_soln_for_github_managed.zip

GitHub Actions Runner - Job 3 (release-to-prod)
├── [Repository Checked Out]
│
├── out/  ← TEMPORARY
│   └── release/
│       └── Model_driven_soln_for_github_managed.zip
│
└── GitHub Artifacts (Downloaded)
    └── managedSolutions/
        └── Model_driven_soln_for_github_managed.zip
```

---

## ❓ FAQ

### Q: Can I see the temporary folders?
**A:** No. Temporary folders (`out/`) exist only during the workflow execution on the GitHub Actions runner. They're not saved anywhere. Once the job completes, the runner is cleaned up.

### Q: Can I download the temporary folder contents?
**A:** Only if you add a step to upload them to artifacts. By default, GitHub only saves artifacts you explicitly upload via `upload-artifact` action.

### Q: Why not just store everything in the repository?
**A:** 
- Large ZIP files (10-50 MB) would bloat the repository
- Artifacts like exported ZIPs are temporary and don't need version control
- GitHub Artifacts are designed for inter-job communication
- Separating temporary and permanent data keeps repositories clean

### Q: Where is the managed ZIP actually stored?
**A:** It's created in two places during workflow:
1. **First time:** `out/solutions/` during Job 1 (temporary)
2. **Uploaded:** GitHub Artifacts for inter-job transfer
3. **Downloaded:** `out/release/` in Jobs 2 & 3 (temporary)
4. **Deployed:** Imported to Power Platform environments

### Q: Can I modify the folder paths?
**A:** Yes! All folder parameters are configurable:
```yaml
solution_source_folder: solutions/          # Can change
solution_outbound_folder: out/solutions/    # Can change
solution_release_folder: out/release        # Can change
```
But the defaults follow best practices.

### Q: What if a temporary folder doesn't exist?
**A:** GitHub Actions automatically creates folders when actions need them. You don't need to manually create them.

### Q: Where is `solutions/` folder located?
**A:** In your Git repository root:
```
/poweractionslab/
├── .github/
│   └── workflows/
├── solutions/              ← HERE
│   └── Model_driven_soln_for_github/
├── README.md
└── WORKFLOWS.md
```

### Q: Can temporary folders contain important data?
**A:** No. All important data is either:
- Stored in Git (`solutions/` folder) - PERSISTENT
- Deployed to Power Platform environments - IN PROD
- Stored as GitHub Artifacts - RECOVERABLE
- Version controlled in pull requests - TRACEABLE

---

## 📊 Folder Summary Table

| Folder | Location | Temporary? | Purpose | Persists? | Access |
|--------|----------|-----------|---------|-----------|--------|
| `solutions/` | Git Repo | ❌ No | Version control | ✅ Yes | GitHub UI, Git clone |
| `out/ship/` | Runner | ✅ Yes | Staging (unused) | ❌ No | Workflow only |
| `out/solutions/` | Runner | ✅ Yes | Pack staging | ❌ No | Workflow only |
| `out/release/` | Runner | ✅ Yes | Deploy staging | ❌ No | Workflow only |
| `out/exported/` | Runner | ✅ Yes | Export staging | ❌ No | Workflow only |
| Artifacts | GitHub Cloud | Semi | Inter-job transfer | ~90 days | GitHub UI |

---

## 🔐 Security Notes

1. **Temporary folders are isolated** - Only accessible within that specific job
2. **Cross-job communication via artifacts** - Secure GitHub Artifacts API
3. **No persistent temporary data** - Can't accidentally commit sensitive files
4. **Clean runner after each job** - Previous job data not accessible in next job
5. **Git repository checked out fresh** - Each job gets clean copy

---

## 📝 Summary

- **Permanent:** `solutions/` folder (version controlled in Git)
- **Temporary:** All `out/` folders (exist only during workflow execution)
- **Inter-job:** Managed via GitHub Artifacts (automatically handled)
- **No manual folder creation needed** - Actions create folders as needed
- **Safe and secure** - Temporary data automatically cleaned up

