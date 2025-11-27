# Knowledge Utility Kit

A project for publishing documentation to GitHub Pages using **MkDocs** and **Material for MkDocs**.

## 📁 Project Structure

```
kukgini.github.io/
├── .github/
│   └── workflows/
│       └── build-and-deploy.yml  # GitHub Actions automated build
├── bin/                    # Build tools (for local use)
│   ├── build              # Build documentation
│   ├── deploy             # Build + Git deployment
│   └── clean              # Clean build artifacts
├── src/                    # Documentation source files (Markdown)
│   ├── ap2/
│   ├── dt/
│   ├── paas/
│   ├── iaas/
│   ├── _static/           # Custom CSS
│   └── index.md           # Home page
├── docs/                   # Build output (for GitHub Pages deployment)
├── mkdocs.yml              # MkDocs configuration
├── requirements.txt        # Python dependencies
└── README.md
```

## 🚀 Build Methods

### 🤖 Automated Build (GitHub Actions)

When you modify and push source files, GitHub Actions will automatically build and deploy the documentation.

**Trigger conditions:**
- Changes to files in the `src/` directory
- Changes to `requirements.txt`
- Push to `main` branch

### 📦 Local Build

#### Prerequisites

- Python 3.x

#### 1. Build Documentation

```bash
./bin/build
```

**Process:**
1. Create an isolated virtual environment (`.venv`)
2. Install dependencies (`mkdocs`, `mkdocs-material`)
3. Build documentation (`src/` → `docs/`)
4. Create `.nojekyll` file
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

#### 3. Clean Build Artifacts

```bash
./bin/clean
```

Deletes the `docs/`, `site/`, and `.venv/` directories.

## 📝 Writing Documentation

### Adding Documents

1. Create a `.md` file in the `src/` directory.
2. Add the new document to `mkdocs.yml` under `nav`:

```yaml
nav:
  - Home: index.md
  - My Section:
    - my-new-doc.md
```

3. Build and deploy.

### Supported Features

- **Markdown**: Standard Markdown syntax.
- **Admonitions**: `!!! note "Title"` blocks.
- **Code Highlighting**: Fenced code blocks with language specifiers.
- **Mermaid Diagrams**:
  ```mermaid
  graph TD
    A[Start] --> B{Is it working?}
    B -- Yes --> C[Great!]
    B -- No --> D[Debug]
  ```

## 🛠️ Configuration

You can change site settings by editing `mkdocs.yml`:

- Change theme colors
- Add plugins
- Modify navigation structure

## 📄 License

This project is a personal documentation repository.
