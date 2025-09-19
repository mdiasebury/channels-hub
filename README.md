# Channels Hub Documentation

## Getting Started

A small change on the readme 
One more change here.

### 1. Clone the Repository
Clone this repository to your local machine:
```bash
git clone git@github.com:mdiasebury/channels-hub.git
cd channels_page
```

### 2. Create and Activate a Virtual Environment
Create a Python virtual environment and activate it:
```bash
python -m venv venv
source venv/bin/activate
```

### 3. Install Dependencies
Install MkDocs Material theme:
```bash
pip install mkdocs-material
```

### 4. Start the Documentation Server
To preview the documentation locally, run:
```bash
mkdocs serve
```
This will start a local server (usually at http://127.0.0.1:8000/) where you can view your docs.

---

## Adding New Documentation Pages

### 1. Add a New Markdown File
- Create your new documentation page as a `.md` file inside the appropriate folder under `docs/` (e.g., `docs/API/`, `docs/EBO/`, `docs/Mobile/`).

### 2. Reference the File in `mkdocs.yml`
- Open `mkdocs.yml` in the root directory.
- Under the `nav:` section, add a new entry for your page. This will add a new tab or section to the navigation bar.

**Example:**
```yaml
nav:
  - Home: index.md
  - API:
      - Overview: API/API.md
      - New Page: API/new-page.md
  - EBO:
      - Overview: EBO/EBO.md
```

- The left side (e.g., `New Page`) is the label that appears in the navigation bar.
- The right side (e.g., `API/new-page.md`) is the path to your markdown file, relative to the `docs/` folder.

### 3. Organize Your Files
- Place each `.md` file in the folder that matches its domain (e.g., API docs in `docs/API/`, EBO docs in `docs/EBO/`).

---

For more details, see the [MkDocs Material documentation](https://squidfunk.github.io/mkdocs-material/).




This is just an exaple of adding something to a readme.
