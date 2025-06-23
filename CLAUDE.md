# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

xword-dl is a Python command-line tool for downloading crossword puzzles from various online outlets and saving them as .puz files. The tool supports 30+ puzzle sources including NYT, Guardian, Washington Post, and many others.

## Development Commands

### Installation and Setup
```bash
# Install in development mode
python setup.py install

# Install dependencies
pip install -r requirements.txt

# Install in virtual environment (recommended)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
python setup.py install
```

### Running the Tool
```bash
# Run directly from source
python -m xword_dl

# After installation
xword-dl

# Common usage examples
xword-dl nyt --latest
xword-dl uni --date "2021-09-22"
xword-dl https://example.com/puzzle-url
```

### Testing
The codebase does not appear to have a formal test suite. Testing is primarily done through manual verification of puzzle downloads.

## Architecture

### Core Components

**Main Entry Point**: `xword_dl/xword_dl.py`
- Contains the main CLI interface and argument parsing
- Orchestrates puzzle downloading via keyword or URL
- Handles authentication flow for subscription sites

**Base Downloader**: `xword_dl/downloader/basedownloader.py`
- Abstract base class for all puzzle downloaders
- Provides common functionality for configuration, filename generation, and HTTP sessions
- Handles authentication tokens and cookies

**Outlet Downloaders**: `xword_dl/downloader/`
- Each outlet has its own downloader class (e.g., `newyorktimesdownloader.py`)
- Inherits from `BaseDownloader`
- Implements outlet-specific puzzle finding and parsing logic
- Some support date-based searching, URL matching, or embedded puzzle detection

**Utilities**: `xword_dl/util/`
- Common utility functions for date parsing, file operations, and configuration
- Exception handling and validation

### Plugin System

The tool uses a plugin-based architecture where each puzzle outlet is implemented as a separate downloader class. The `get_plugins()` function dynamically discovers all available downloaders.

Key plugin capabilities:
- `command`: CLI keyword (e.g., "nyt", "wp")
- `matches_url()`: Can handle direct puzzle URLs
- `matches_embed_url()`: Can extract puzzles from embedded solvers
- `find_by_date()`: Supports date-based puzzle selection
- `authenticate()`: Handles subscription authentication

### Configuration

The tool uses YAML configuration files stored at `~/.config/xword-dl/xword-dl.yaml`. Configuration supports:
- Per-outlet filename templates
- Authentication tokens
- Custom headers and cookies
- Output formatting options

### Data Flow

1. CLI arguments are parsed in `main()`
2. Either `by_keyword()` or `by_url()` is called
3. Appropriate downloader plugin is selected
4. Downloader finds the puzzle URL (latest, by date, or provided)
5. Puzzle is downloaded and parsed using the `puzpy` library
6. Puzzle is saved as a .puz file with generated filename

## Key Dependencies

- `puzpy`: For .puz file format handling
- `beautifulsoup4`: HTML parsing
- `requests`: HTTP requests
- `dateparser`: Date parsing
- `pyyaml`: Configuration files
- `html2text`: HTML to markdown conversion

## Adding New Outlets

To add a new puzzle outlet:

1. Create a new downloader class in `xword_dl/downloader/`
2. Inherit from `BaseDownloader`
3. Set class attributes: `command`, `outlet`, `outlet_prefix`
4. Implement required methods:
   - `find_latest()`: Find the most recent puzzle URL
   - `download(url)`: Download and parse puzzle from URL
   - Optional: `find_by_date()`, `matches_url()`, `authenticate()`

The plugin will be automatically discovered via the `get_plugins()` function.