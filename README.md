# Blog

Personal blog built with MkDocs Material.

## Setup

This project uses a local Python virtual environment to keep all dependencies isolated to this folder.

### Prerequisites

- Python 3.8 or higher

### Installation

1. **Activate the virtual environment:**

   ```bash
   # On macOS/Linux
   source .venv/bin/activate
   
   # On Windows
   .venv\Scripts\activate
   ```

2. **Install dependencies:**

   ```bash
   pip install -r requirements.txt
   ```

### Usage

**Serve locally (with hot reload):**
```bash
mkdocs serve
```

**Build the site:**
```bash
mkdocs build
```

**Deactivate virtual environment when done:**
```bash
deactivate
```

## Notes

- The `.venv` directory contains all Python dependencies and is already in `.gitignore`
- All dependencies are installed locally in this folder, not system-wide
- Always activate the virtual environment before running MkDocs commands
