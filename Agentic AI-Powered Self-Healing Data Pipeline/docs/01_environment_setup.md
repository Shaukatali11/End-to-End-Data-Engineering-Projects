# Agentic AI Self-Healing Data Pipeline
## 01. Project Setup & Environment Configuration

This document contains the complete development environment setup followed for the Agentic AI Self-Healing Data Pipeline project.
---

## 1. Project Overview

**Project:** Agentic AI-Powered Self-Healing Data Pipeline

**Objective:** Build an intelligent data pipeline that can monitor execution, detect failures, investigate root causes using AI agents, recommend corrective actions, and safely recover failed pipeline stages.

### Initial Technology Stack

| Component | Technology |
|---|---|
| IDE | Visual Studio Code |
| Programming Language | Python |
| Data Processing | PySpark |
| Java Runtime | OpenJDK 21 |
| Local Database | SQLite |
| Data Storage | Parquet / Delta Lake |
| Version Control | Git |
| Operating System | Windows |
| Python Environment | Virtual Environment (.venv) |

---

## 2. Create Project Directory

Open VS Code and launch the PowerShell terminal.

Navigate to the desired project location.

```powershell
cd "C:\....\Agentic_AI"
```

### Create the complete folder structure

Execute the following single PowerShell script:

```powershell
$root = "agentic-self-healing-data-pipeline"

$folders = @(
    "configs",
    "data/source",
    "data/raw",
    "data/processed",
    "data/sample_failures",
    "data/database",
    "notebooks",
    "src/ingestion",
    "src/transformations",
    "src/validation",
    "src/monitoring",
    "src/audit",
    "src/agents",
    "src/orchestration",
    "src/recovery",
    "src/llm",
    "src/knowledge_base",
    "src/utils",
    "tests",
    "deployment",
    "docs",
    "logs"
)

New-Item -ItemType Directory -Path $root -Force | Out-Null

foreach ($folder in $folders) {
    New-Item -ItemType Directory -Path "$root/$folder" -Force | Out-Null
}

$files = @(
    "README.md",
    "requirements.txt",
    ".env",
    ".env.example",
    ".gitignore",
    "configs/config.yaml",
    "configs/pipeline_config.yaml",
    "configs/agent_config.yaml",
    "src/__init__.py",
    "src/ingestion/__init__.py",
    "src/transformations/__init__.py",
    "src/validation/__init__.py",
    "src/monitoring/__init__.py",
    "src/audit/__init__.py",
    "src/agents/__init__.py",
    "src/orchestration/__init__.py",
    "src/recovery/__init__.py",
    "src/llm/__init__.py",
    "src/knowledge_base/__init__.py",
    "src/utils/__init__.py"
)

foreach ($file in $files) {
    New-Item -ItemType File -Path "$root/$file" -Force | Out-Null
}

Set-Content "$root/.gitignore" @"
venv/
.env
__pycache__/
*.pyc
.pytest_cache/
.ipynb_checkpoints/
.sparkStaging/
data/raw/*
data/processed/*
data/database/*
logs/*
"@

Set-Location $root

Write-Host "Project folder structure created successfully!"
```

Open the project in VS Code:

```powershell
code .
```

### Folder Structure

```text
agentic-self-healing-data-pipeline/
│
├── configs/
├── data/
│   ├── source/
│   ├── raw/
│   ├── processed/
│   ├── sample_failures/
│   └── database/
│
├── notebooks/
│
├── src/
│   ├── ingestion/
│   ├── transformations/
│   ├── validation/
│   ├── monitoring/
│   ├── audit/
│   ├── agents/
│   ├── orchestration/
│   ├── recovery/
│   ├── llm/
│   ├── knowledge_base/
│   └── utils/
│
├── tests/
├── deployment/
├── docs/
├── logs/
│
├── README.md
├── requirements.txt
├── .env
├── .env.example
└── .gitignore
```

---

## 3. Check Existing Software Versions

Before installing project dependencies, check the software already installed on the system.

### 3.1 Python Version

```powershell
python --version
```

Output:

```text
Python 3.12.0
```

### 3.2 Python Installation Location

```powershell
where.exe python
```

**Decision:** Use Python 3.12 explicitly for this project to avoid conflicts with other installations.

### 3.3 pip Version

```powershell
python -m pip --version
```

Output:

```text
pip 24.2
```

### 3.4 Java Version

```powershell
java -version
```

Initially detected:

```text
java version "24.0.1"
```

### 3.5 Git Version

```powershell
git --version
```

Output:

```text
git version 2.38.1.windows.1
```

### 3.6 SQLite Version

SQLite is included with Python through the `sqlite3` module.

```powershell
python -c "import sqlite3; print('SQLite version:', sqlite3.sqlite_version)"
```

Output:

```text
SQLite version: 3.42.0
```

---

## 4. Install and Configure Java 21

PySpark requires a compatible Java runtime.

For a stable development environment, we selected Java 21 LTS.

### 4.1 Install Java 21

We then used Microsoft OpenJDK 21:

```powershell
winget install --id Microsoft.OpenJDK.21 --exact
```

Installation directory detected:

```text
C:\Program Files\Microsoft\jdk-21.0.13.102-hotspot
```

### 4.2 Configure JAVA_HOME for the current terminal

```powershell
$env:JAVA_HOME = "C:\Program Files\Microsoft\jdk-21.0.13.102-hotspot"

$env:Path = "$env:JAVA_HOME\bin;$env:Path"
```

### 4.3 Configure JAVA_HOME permanently

```powershell
[Environment]::SetEnvironmentVariable(
    "JAVA_HOME",
    "C:\Program Files\Microsoft\jdk-21.0.13.102-hotspot",
    "User"
)
```

Add the following entry to the Windows User Path:

```text
%JAVA_HOME%\bin
```

Move it above older Java entries if necessary.

Restart VS Code after making permanent environment changes.

### 4.4 Verify Java

```powershell
java -version

javac -version

echo $env:JAVA_HOME
```

Expected:

```text
Java version: 21.0.13

JAVA_HOME:
C:\Program Files\Microsoft\jdk-21.0.13.102-hotspot
```

**Important:** The machine originally had Java 24 and Java 18 configured. We selected Java 21 for this project.

---

## 5. Create Python Virtual Environment

A virtual environment isolates project dependencies from global Python and Anaconda installations.

### 5.1 Create the environment

Execute from the project root:

```powershell
& "C:\..\....\....\...\Programs\Python\Python312\python.exe" -m venv .venv
```

This creates:

```text
.venv/
```

### 5.2 Activate the environment

```powershell
.\.venv\Scripts\Activate.ps1
```

Expected terminal:

```text
(.venv) PS C:\data agent\Agentic_AI\agentic-self-healing-data-pipeline>
```

### 5.3 PowerShell execution policy issue

If PowerShell blocks activation:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then activate again:

```powershell
.\.venv\Scripts\Activate.ps1
```

This changes the policy only for the current PowerShell process.

### 5.4 Verify Python environment

```powershell
python --version

python -m pip --version

where.exe python
```

Expected first Python location:

```text
agentic-self-healing-data-pipeline\.venv\Scripts\python.exe
```

Python version:

```text
Python 3.12.0
```

---

## 6. Upgrade pip

Upgrade pip inside the virtual environment:

```powershell
python -m pip install --upgrade pip
```

Verify:

```powershell
python -m pip --version
```

---

## 7. Install PySpark

We selected PySpark 4.0.1 for our initial development environment.

Install:

```powershell
python -m pip install pyspark==4.0.1
```

Verify:

```powershell
python -c "import pyspark; print('PySpark version:', pyspark.__version__)"
```

Expected:

```text
PySpark version: 4.0.1
```

---

## 8. Install Supporting Python Libraries

Install the initial supporting dependencies:

```powershell
python -m pip install pandas pyarrow pyyaml python-dotenv pytest
```

Purpose of each library:

| Library | Purpose |
|---|---|
| pandas | Data analysis and manipulation |
| pyarrow | Columnar data processing and Parquet support |
| pyyaml | Read YAML configuration files |
| python-dotenv | Load environment variables from `.env` |
| pytest | Automated testing |
| pyspark | Distributed data processing |

Additional libraries such as LangGraph, LangChain, OpenAI, ChromaDB, Delta Lake and FastAPI will be installed during their respective implementation phases.

---

## 9. Environment Summary

| Component | Selected Version / Tool |
|---|---|
| OS | Windows |
| IDE | VS Code |
| Python | 3.12.0 |
| pip | Initially 24.2 |
| Java | OpenJDK 21.0.13 |
| PySpark | 4.0.1 |
| Git | 2.38.1 |
| SQLite | 3.42.0 |
| Virtual Environment | `.venv` |

---

## 10. Important Development Practices

1. Always activate `.venv` before installing dependencies or executing project code.
2. Use `python -m pip` instead of global `pip` commands.
3. Keep API keys and credentials inside `.env`.
4. Never push `.env` or sensitive credentials to GitHub.
5. Maintain dependencies in `requirements.txt`.
6. Keep ingestion, transformations, monitoring and AI agents in separate modules.
7. Test individual components before integrating the complete system.
8. Maintain documentation for every development phase.

---
