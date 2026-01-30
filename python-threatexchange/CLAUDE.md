# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`python-threatexchange` is a Python library and CLI tool for simplifying the exchange of trust & safety signals (especially media hash exchanges). It provides a pluggable architecture for implementing signal types (like PDQ photo hashing), content types (photo, video, text), and signal exchange APIs (ThreatExchange, NCMEC, StopNCII, etc.).

## Development Commands

### Setup and Installation
```bash
# Install in editable mode with dev dependencies
pip install -e '.[dev]'

# Install with all extensions (optional dependencies like vPDQ, TLSH, PDF)
pip install -e '.[all]'
```

### Testing
```bash
# Run all tests
pytest

# Run specific test file
pytest threatexchange/signal_type/tests/test_signal_type.py

# Run tests for a specific module
pytest threatexchange/cli/tests/
```

### Code Quality
```bash
# Format code with black
black .

# Type checking with mypy
mypy threatexchange

# Before submitting PR, run all three:
black . && mypy threatexchange && pytest
```

### CLI Usage
```bash
# The CLI is available as 'threatexchange' or 'tx'
threatexchange --help

# Common workflow:
threatexchange fetch              # Download signals from configured APIs
threatexchange match photo img.jpg  # Match content against indices
threatexchange hash photo img.jpg   # Generate hashes for content
threatexchange dataset             # View stored signals

# CLI state is stored in ~/.threatexchange
# Factory reset:
threatexchange --factory-reset
```

## Core Architecture

The library is built around four key abstractions that work together:

### 1. SignalType (`threatexchange/signal_type/signal_base.py`)

Represents a technique for generating signatures/hashes of content that can be shared and matched.

**Key characteristics:**
- Must be serializable as a string (for easy API exchange)
- Implements both naive/brute-force matching (for correctness) and optimized matching via Index
- Associates with one or more ContentTypes it applies to
- Examples: `PdqSignal` (photo hashing), `VideoMD5Signal`, `RawTextSignal`, `TrendQuerySignal`

**Required methods:**
- `get_content_types()` - Which content types this signal applies to
- `get_index_cls()` - Returns the Index class for efficient matching
- `validate_signal_str()` - Normalizes/validates signal strings
- `compare_hash()` or similar matching method

**Common base classes:**
- `SimpleSignalType` - For basic hash-based signals
- `BytesHasher` - For signals that hash raw bytes
- `FileHasher` - For signals that hash files

### 2. ContentType (`threatexchange/content_type/content_base.py`)

Represents a type of content (photo, video, text, URL) that can have signals generated from it.

**Key characteristics:**
- Minimal interface - mainly for organizing which signals apply to which content
- Can extract additional content (e.g., URL → download page → extract photos/text)
- Examples: `PhotoContent`, `VideoContent`, `TextContent`, `URLContent`

**Optional methods:**
- `extract_additional_content()` - Post-process to find embedded content

### 3. SignalExchangeAPI (`threatexchange/exchanges/signal_exchange_api.py`)

Defines how to fetch and potentially write signals from an external API or data source.

**Key characteristics:**
- Handles authentication and API-specific details
- Implements checkpoint-able fetching (for incremental updates)
- Converts API format to library's signal format
- Supports both full fetches and delta updates

**Core methods:**
- `fetch_iter()` - Yield update records with checkpointing
- `naive_convert_to_signal_type()` - Convert API format to signals
- `for_collab()` - Factory method that discovers credentials

**Implementations:**
- `FBThreatExchangeSignalExchangeAPI` - Meta's ThreatExchange platform
- `NCMECSignalExchangeAPI` - NCMEC hash database
- `StopNCIISignalExchangeAPI` - StopNCII.org
- `TATSignalExchangeAPI` - TechAgainstTerrorism
- `LocalFileSignalExchangeAPI` - Local files for testing
- `StaticSampleSignalExchangeAPI` - Built-in sample data

### 4. SignalTypeIndex (`threatexchange/signal_type/index.py`)

Provides efficient matching of signals at scale, building optimized data structures.

**Key characteristics:**
- Stores signals in memory-efficient structures (e.g., FAISS for PDQ)
- Implements `query()` method that returns potential matches with distance/similarity
- Three common base implementations:
  - `TrivialSignalTypeIndex` - Exact match only (like MD5)
  - `TrivialLinearSearchHashIndex` - Linear scan using compare_hash
  - `TrivialLinearSearchMatchIndex` - For regex/pattern matching

**Example:** `PDQIndex` uses FAISS for approximate nearest neighbor search on PDQ hashes

## Data Flow

1. **Configure collaborations** - Define which signal sources to use
2. **Fetch** - Download signals from APIs via SignalExchangeAPI
3. **Store** - Save raw API responses and convert to signals
4. **Build indices** - Create optimized Index structures for each SignalType
5. **Match** - Hash content → query indices → return matches

## Extensions System

Extensions allow adding new SignalTypes, ContentTypes, and APIs without modifying core code.

**Creating an extension:**
1. Implement your SignalType/ContentType/API classes
2. Create a `TX_MANIFEST` variable with a `ThreatExchangeExtensionManifest`
3. Install and add via: `threatexchange config extensions add your.module`

**Built-in extensions** (require extra dependencies):
- `vpdq` - Video hashing using vPDQ
- `tlsh` - TLSH fuzzy hashing for text
- `pdq_ocr` - PDQ + OCR for meme detection
- `pdf` - PDF content extraction

See `threatexchange/extensions/README.md` for details.

## Important Patterns

### Signal Distance and Matching
- Many signals support "distance" or similarity (e.g., PDQ's Hamming distance)
- `SignalComparisonResult` wraps match result + distance info
- `SignalSimilarityInfo` provides context about matches (used in logging/debugging)
- Thresholds are configurable per collaboration

### Storage Layer
- CLI stores data in `~/.threatexchange/`
- Storage implementations in `threatexchange/storage/`
- `local_dbm.py` - Simple DBM-based storage
- Supports both full dataset and incremental updates

### CLI Command Structure
- Commands in `threatexchange/cli/*_cmd.py`
- All extend `command_base.Command`
- State managed via `CLiConfig` and `CliState`
- Settings in `CLISettings` (loaded from `~/.threatexchange/`)

### Authentication
- Credentials discovered from environment or config files
- Common patterns:
  - Environment variables: `TX_ACCESS_TOKEN`, `TX_NCMEC_CREDENTIALS`, etc.
  - Config files: `~/.txtoken`, `~/.tx_stopncii_keys`
  - CLI config: `threatexchange config api <api_name> --credentials ...`

### Type Safety
- Mypy type checking is a work in progress
- New modules should aim for `--strict` compliance
- Add module to `mypy.ini` with `strict = True`
- Use `# type: ignore[error-code]` sparingly with specific error codes

## Testing Patterns

### SignalType Testing
- Use `signal_type_test_helper.py` for validating SignalType implementations
- Ensures correct implementation of required methods
- Tests serialization, comparison, and index behavior

### Extension Testing
- Many tests are skipped if optional dependencies aren't installed
- Install extension deps to run their tests locally
- CI runs all tests with all dependencies

## File Organization

```
threatexchange/
├── cli/                    # CLI commands and config
│   ├── dataset/            # Dataset-related subcommands
│   ├── *_cmd.py           # Individual command implementations
│   └── cli_config.py      # CLI state and settings management
├── content_type/          # Content type implementations
│   └── preprocess/        # Content preprocessing utilities
├── exchanges/             # Signal exchange API implementations
│   ├── clients/           # API client helpers
│   └── impl/              # Concrete API implementations
├── extensions/            # Optional extensions (vPDQ, TLSH, PDF, etc.)
├── signal_type/           # Signal type implementations
│   ├── pdq/               # PDQ implementation with FAISS index
│   └── tests/             # Signal type testing helpers
├── storage/               # Data storage abstractions
└── tests/                 # Integration tests
```

## PDQ Implementation Deep Dive

PDQ (Photo DNA Quality) is the most complex signal type, serving as a reference implementation:

- **Signal:** `threatexchange/signal_type/pdq/signal.py` - PdqSignal class
- **Hasher:** `pdq_hasher.py` - Wraps pdqhash library, handles quality threshold
- **Index:** `pdq_index.py` + `pdq_index2.py` - Linear and FAISS-based implementations
- **Matcher:** `pdq_faiss_matcher.py` - FAISS integration for fast approximate matching
- **Utils:** `pdq_utils.py` - Distance calculation, thresholds

**Key insight:** PDQ has both a simple linear index and a FAISS-backed index. The library automatically chooses based on dataset size and available resources.

## Common Development Tasks

### Adding a new SignalType
1. Extend `SignalType` in `threatexchange/signal_type/`
2. Implement required methods: `get_content_types()`, `get_index_cls()`, `compare_hash()`
3. Create an Index class (can start with `TrivialLinearSearchHashIndex`)
4. Add to `_DEFAULT_SIGNAL_TYPES` in `cli/main.py` (or create as extension)
5. Add tests using `signal_type_test_helper`

### Adding a new SignalExchangeAPI
1. Extend `SignalExchangeAPI` in `threatexchange/exchanges/impl/`
2. Implement `fetch_iter()` for incremental fetching with checkpoints
3. Implement `naive_convert_to_signal_type()` to convert API data to signals
4. Add credential handling (env vars, config files, etc.)
5. Register in `cli/main.py` or as extension

### Debugging Index Build Performance
- Index build happens in `threatexchange/cli/fetch_cmd.py` after fetching
- Use `--verbose` flag for timing info
- Large datasets may require FAISS or other optimized indices
- Check `SignalTypeIndex` implementations for memory usage patterns
