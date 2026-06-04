# qobuz-dl Agent Notes

## Quick Reference

- **Entry point**: `qobuz_dl/__init__.py:main` → `qobuz_dl/cli.py:main()`
- **Main class**: `QobuzDL` in `qobuz_dl/core.py`
- **CLI commands**: `fun` (interactive), `dl` (download by URL), `lucky` (search & download)

## Key Files

- `qobuz_dl/cli.py`: CLI entry, config file handling (`~/.config/qobuz-dl/config.ini`)
- `qobuz_dl/core.py`: `QobuzDL` class with download, search, interactive modes
- `qobuz_dl/qopy.py`: Qobuz API client wrapper (from Qo-DL-Reborn, Sorrow446)
- `qobuz_dl/bundle.py`: Fetches app_id and secrets from Qobuz web app
- `qobuz_dl/downloader.py`: Actual download + FLAC/MP3 tagging via mutagen
- `qobuz_dl/utils.py`: `get_url_info()` parses Qobuz URLs, `smart_discography_filter()` filters artist discographies

## Style

- **flake8** (`.flake8`): max-line-length 80, ignore E203, E266, E501
- **No type hints** in this codebase
- **No tests** currently exist

## Quirks

- Passwords are stored as MD5 hashes in config
- `pick==1.6.0` (pin locked for curses compatibility on Windows)
- `qopy.py` and `spoofer` are borrowed from discontinued Qo-DL-Reborn
