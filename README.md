neoEnigma Classic – Offline Cryptanalysis Toolkit

neoEnigma is a terminal-based cryptanalysis toolkit for classical ciphers, focused on:
	•	monoalphabetic substitution ciphers
	•	Vigenere ciphers

It is designed to run fully offline on a local Linux (Ubuntu) system and ships with an offline bootstrap workflow, structured logging, tests, and an optional lightweight CI pipeline.

⸻

Features
	•	High-speed n-gram scoring engine
	•	Trigram and quadgram language models
	•	Dense array backing, with optional NumPy acceleration
	•	Monoalphabetic solver
	•	Simulated annealing with multiple restarts
	•	Frequency-seeded initial keys (ETAOIN SHRDLU)
	•	Rich neighborhood moves (swaps and 3-cycles)
	•	Vigenere solver
	•	Index-of-coincidence based key length detection
	•	Column-wise correlation against English letter frequencies
	•	Key refinement using n-gram scoring
	•	Offline-first design
	•	No network access; all data is local
	•	Suitable for air-gapped and constrained environments
	•	Developer-friendly architecture
	•	Typed configuration (dataclasses)
	•	Structured JSON logging
	•	Unit and integration tests
	•	Offline CI helper script

⸻

Repository Layout

neoEnigma/
  api/
    cli.py                # Main entrypoint / CLI
  core/
    text.py               # Normalization and text <-> int utilities
    ngram_loader.py       # Load n-gram CSVs into arrays
    scoring.py            # N-gram scoring engine
  infra/
    config.py             # AppConfig dataclass and constants
    logging_local.py      # JSON logging setup
  services/
    mono_solver.py        # Monoalphabetic SA solver
    vigenere_solver.py    # Vigenere solver
  scripts/
    offline_ci.sh         # Offline CI (lint, type-check, tests)
  tests/
    test_scoring.py       # Scorer tests
    test_mono.py          # Mono solver tests
    test_vigenere.py      # Vigenere solver tests
  data/                   # (You) provide n-gram CSV files here
  inputs/                 # (You) provide ciphertext files here
  logs/                   # JSON log files (optional)
  ui/
    ops_report.md         # Notes on operational / UI design
  run_examples.sh         # Small self-contained demo script
  README.md
  ARCHITECTURE.md
  SECURITY.md
  LICENSE
  pyproject.toml
  .gitignore

If you are using the bootstrap_offline.sh script on a fresh Ubuntu machine, it will create the neoEnigma/ tree, write all files, set executable bits, optionally create a virtualenv, and run the offline CI script.

⸻

Requirements
	•	Python 3.10 or newer
	•	Linux (tested with Ubuntu)
	•	Optional but recommended:
	•	NumPy (for faster scoring and correlation)
	•	pytest (for tests)
	•	flake8 (lint)
	•	mypy (static type checking)

The project is designed to work without network access. You are responsible for provisioning Python and any Python packages locally (from offline media, etc).

⸻

Data Files

neoEnigma expects n-gram frequency files in CSV format:
	•	Trigrams (3-grams)
	•	Quadgrams (4-grams)

Place them into data/ and point the CLI at them.

Example filenames:
	•	data/english_trigrams.csv
	•	data/english_quadgrams.csv

Expected CSV format:

GRAM,COUNT
THE,12345
AND,6789
...

Notes:
	•	Header row is optional; if present it must be something like GRAM,COUNT.
	•	GRAM values must be uppercase A to Z, with length equal to the n-gram size (3 or 4).
	•	COUNT is an integer frequency.

⸻

Ciphertext Input

Ciphertext files are plain text files, for example:

QEBNRFZHYLUXJPALTKH...

During processing:
	•	Text is uppercased.
	•	All non A-Z characters are stripped.
	•	Only the resulting A-Z sequence is used for scoring and solving.

Place your ciphertext into inputs/ and pass the file path to --cipher-file.

⸻

Quick Start
	1.	Clone the repository (or generate it via the offline bootstrap script).
	2.	Ensure Python 3.10+ is installed.
	3.	Create and activate a virtualenv (optional but recommended).
	4.	Provide n-gram CSV files in data/.
	5.	Provide ciphertext files in inputs/.

Example (with virtualenv):

python3 -m venv .venv
source .venv/bin/activate

pip install numpy pytest flake8 mypy

Prepare data:

mkdir -p data inputs
cp /path/to/english_trigrams.csv data/
cp /path/to/english_quadgrams.csv data/
cp /path/to/my_cipher.txt inputs/


⸻

CLI Usage

The main entrypoint is api/cli.py.

python3 api/cli.py --help

Key arguments:
	•	--cipher-file (required)
Path to ciphertext file.
	•	--mode
mono (default) or vigenere.
	•	--trigram-file (required)
Trigram CSV path.
	•	--quadgram-file (required)
Quadgram CSV path.
	•	--seed
RNG seed for reproducible runs (defaults to current timestamp).
	•	--disable-ui
Disable banner / extra printing for batch mode.
	•	--log-file
Path to JSON log file (if omitted, logs go to stdout).

Scoring-related:
	•	--weights-tri
Weight for trigram component (default 0.3).
	•	--weights-quad
Weight for quadgram component (default 0.7).
	•	--entropy-threshold
Default entropy threshold (currently informational; adjustable).

Mono solver specific:
	•	--restarts
Number of simulated annealing restarts (default 64).
	•	--iterations
SA iterations per restart (default 5000).
	•	--sa-T0
Initial temperature (default 10.0).
	•	--sa-alpha
Temperature decay factor (default 0.9995).

Vigenere solver specific:
	•	--vigenere-max-keylen
Maximum key length to test (default 16).
	•	--vigenere-multilen
Evaluate multiple candidate key lengths (optional flag).
	•	--topk-per-column
Number of candidate shifts per column (default 3).

⸻

Monoalphabetic Example

python3 api/cli.py \
  --mode mono \
  --cipher-file inputs/my_mono_cipher.txt \
  --trigram-file data/english_trigrams.csv \
  --quadgram-file data/english_quadgrams.csv \
  --restarts 64 \
  --iterations 10000 \
  --sa-T0 15.0 \
  --sa-alpha 0.9998 \
  --seed 1337

Example output (simplified):

--- Decryption Complete ---
Mode              : mono
Best Score (norm) : -2.3456
Recovered Key     : QWERTYUIOPASDFGHJKLZXCVBNM
Runtime           : 12.34 seconds

--- Plaintext Preview ---
THISISANEXAMPLEOFDECRYPTEDTEXT...
-------------------------


⸻

Vigenere Example

python3 api/cli.py \
  --mode vigenere \
  --cipher-file inputs/my_vigenere_cipher.txt \
  --trigram-file data/english_trigrams.csv \
  --quadgram-file data/english_quadgrams.csv \
  --vigenere-max-keylen 20 \
  --topk-per-column 3 \
  --seed 1337

The Vigenere solver will:
	1.	Estimate a likely key length using index of coincidence.
	2.	For each column, score candidate shifts against English frequencies.
	3.	Build an initial key from best shifts.
	4.	Refine the key using the n-gram scorer.
	5.	Print the best key and plaintext preview.

⸻

Demo Script

The repository includes a small offline demo script:

bash ./run_examples.sh

This script:
	•	Creates tiny dummy n-gram files in data/.
	•	Writes minimal example ciphertexts into inputs/.
	•	Runs both the mono and Vigenere solvers with those examples.
	•	Prints results and then cleans up the temporary files.

This is useful for verifying that the toolkit runs end-to-end without needing external data.

⸻

Offline CI and Tests

There is a simple offline CI helper script at scripts/offline_ci.sh.

Run it from the project root:

bash scripts/offline_ci.sh

This will:
	1.	Run flake8 (if available) for basic linting.
	2.	Run mypy (if available) for static type checking.
	3.	Run pytest with coverage over tests/.

Notes:
	•	If flake8 or mypy are not installed, the script will print a warning and continue.
	•	If pytest is missing, the script will fail (tests are required).

You can also run tests directly:

pytest
pytest -q tests/test_mono.py
pytest -q tests/test_vigenere.py


⸻

Logging and Observability

Logging is configured by infra/logging_local.py:
	•	Logs are JSON-formatted.
	•	By default, logs are emitted to stdout.
	•	If --log-file is provided, logs go to a rotating file in logs/.

Example log entry:

{
  "timestamp": "2025-01-01 12:34:56",
  "level": "INFO",
  "message": "Run finished. Final metrics: { ... }",
  "module": "root"
}

Use these logs to:
	•	Inspect solver runs.
	•	Capture metrics for batch jobs.
	•	Integrate with external tooling in offline pipelines.

⸻

Architecture and Design

High level design goals:
	•	Strict separation of concerns:
	•	infra for config/logging.
	•	core for low-level primitives and scoring.
	•	services for cryptanalytic algorithms.
	•	api for public, user-facing entrypoints.
	•	Fast, deterministic scoring:
	•	Integer-encoded text.
	•	Rolling hash indexing into contiguous n-gram arrays.
	•	Offline-safe:
	•	No network calls.
	•	Minimal dependencies.

For more details, see:
	•	ARCHITECTURE.md for a deeper walkthrough of the module layout and data flow.
	•	ui/ops_report.md for notes on how to integrate neoEnigma into larger toolchains or frontends.

⸻

Security Considerations

This project is intentionally designed for offline, local use. Key points:
	•	No network access
	•	No secrets management
	•	Local files only (ciphertext, n-grams, logs)
	•	ASCII-focused I/O for simplicity

See SECURITY.md for the full security posture and usage guidance.

⸻

License

This project is licensed under the MIT License. See the LICENSE file for details.

⸻

Status

neoEnigma is intended as a practical, offline-friendly cryptanalysis toolkit and a reference implementation of:
	•	classic n-gram scoring
	•	simulated annealing over monoalphabetic keys
	•	Vigenere key-length estimation and refinement

Contributions and extensions (new scoring models, additional classical ciphers, better heuristics) are welcome via pull requests.
