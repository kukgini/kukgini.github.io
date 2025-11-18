# my-sphinx

A project for publishing documentation to GitHub Pages using Sphinx.

## 📁 Project Structure

```
my-sphinx/
├── .github/
│   └── workflows/
│       └── build-and-deploy.yml  # GitHub Actions automated build
├── bin/                    # Build tools (for local use)
│   ├── build              # Build documentation
│   ├── deploy             # Build + Git deployment
│   └── clean              # Clean build artifacts
├── src/                    # Documentation source files
│   ├── bosh/
│   ├── cf/
│   ├── git/
│   ├── golang/
│   ├── os/
│   ├── md/
│   ├── z-diagrams/
│   ├── conf.py            # Sphinx configuration
│   └── index.rst          # Documentation main page
├── docs/                   # Build output (for GitHub Pages deployment)
│   ├── .nojekyll          # Disable Jekyll for GitHub Pages
│   ├── index.html         # Main HTML page
│   ├── _static/           # Static files (CSS, JS, etc.)
│   └── ...                # Other HTML files
├── requirements.txt        # Python dependencies
└── README.md
```

## 🚀 Build Methods

### 🤖 Automated Build (GitHub Actions)

When you modify and commit source files in the `src/` directory, GitHub Actions will automatically build and deploy the documentation.

**Trigger conditions:**
- Changes to files in the `src/` directory
- Changes to `requirements.txt`
- Push to `master` or `main` branch

**GitHub Actions execution status:**
- Check execution status in the repository's **Actions** tab
- After build completion, the `docs/` directory is automatically updated

### 📦 Local Build

#### Prerequisites

- Python 3.x

#### 1. Build Documentation

```bash
./bin/build
```

**Process:**
1. Create an isolated virtual environment (`.venv`)
2. Install dependencies (`sphinx`, `sphinx_rtd_theme`, `recommonmark`)
3. Build HTML documentation with Sphinx (`src/` → `docs/`)
4. Create `.nojekyll` file (for GitHub Pages)
5. Automatically clean up virtual environment

Build output is generated in the `docs/` directory.

#### 2. Build + GitHub Pages Deployment

```bash
./bin/deploy
```

**Process:**
1. Execute documentation build
2. Git add changes
3. Automatic commit
4. Push to GitHub

> **💡 Tip**: With GitHub Actions, you only need to modify sources and push without building locally!

#### 3. Clean Build Artifacts

```bash
./bin/clean
```

Deletes the `docs/`, `_build/`, and `.venv/` directories.

## 📝 Writing Documentation

### Adding Documents

1. Create a `.rst` or `.md` file in the `src/` directory
2. Add the new document to `src/index.rst`:

```rst
.. toctree::
   :maxdepth: 1

   your-new-document.md
```

3. Build and deploy:

```bash
./bin/deploy
```

### Supported Formats

- **reStructuredText** (`.rst`): Sphinx default format
- **Markdown** (`.md`): Supported through the `recommonmark` extension

## 🌐 GitHub Pages Setup

### 1. Enable GitHub Pages

1. GitHub repository → **Settings** → **Pages**
2. **Source**: Deploy from a branch
3. **Branch**: `master` (or `main`)
4. **Folder**: Select `/docs`
5. **Save**

### 2. GitHub Actions Permissions (Important!)

1. GitHub repository → **Settings** → **Actions** → **General**
2. In the **Workflow permissions** section:
   - Select **Read and write permissions**
   - Check **Allow GitHub Actions to create and approve pull requests**
3. **Save**

Without this setting, GitHub Actions cannot commit build results.

### 3. Verify Automated Deployment

Now when you modify and push files in the `src/` directory:

1. **GitHub Actions** automatically executes the build
2. Build results are committed to `docs/`
3. **GitHub Pages** automatically updates

**Check execution:**
- Check workflow execution status in the repository's **Actions** tab
- When complete, the GitHub Pages site automatically updates

## 🛠️ Advanced Usage

### Manual Python Execution

```bash
# Build
python3 bin/build

# Deploy
python3 bin/deploy

# Clean
python3 bin/clean
```

### Complete Rebuild

```bash
./bin/clean
./bin/build
```

### Changing Sphinx Configuration

You can change Sphinx settings by editing the `src/conf.py` file:

- Change theme
- Add extensions
- Modify project metadata

## 📦 Dependencies

The project uses the following Python packages:

- **sphinx**: Documentation generation tool
- **sphinx_rtd_theme**: Read the Docs theme
- **recommonmark**: Markdown support

Dependencies are automatically installed in an isolated virtual environment during build, so they don't affect your system Python environment.

## 🔒 Isolated Build Environment

All builds run in a temporary virtual environment:

1. Create `.venv` when build starts
2. Install dependencies
3. Build documentation
4. Automatically delete `.venv` after build completes

You can use all necessary build tools while keeping your system Python environment clean.

## 📄 License

This project is a personal documentation repository.
