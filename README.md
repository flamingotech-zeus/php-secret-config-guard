# PHP Secret Config Guard

[English](README.md) | [简体中文](README.zh-CN.md)

> Store credentials safely in a PHP project, and avoid the one trap that silently truncates an entire file.

## What this is

> **Note on de-identification.** This README is written generically. The skill file itself may still reference internal domains, host names or local absolute paths. Run a scan pass (see `opensource-preflight-audit`) and replace them before publishing.

There are two halves here, and the second one is the reason the skill exists.

The first half is ordinary: keep credentials out of the web root, out of version control, and out of anywhere a request can reach. The second half is a specific trap — a closing PHP tag inside a comment terminates the script. Everything after it is emitted as plain text to the client, which means your credentials, your configuration and the rest of the file are printed to the browser.

The symptom looks nothing like a syntax problem. Every PHP endpoint starts returning source fragments at once, and the usual instinct is to look for a deployment or caching fault.

## Why we open-sourced it

We hit this once. The diagnosis took longer than the fix, which is the usual ratio for traps like this and the reason it is worth writing down.

It is a portable piece of PHP operational knowledge with nothing of ours in it. Anyone running PHP behind a panel-based host will eventually be one careless keystroke away from it.

## What it does

- **Where credentials belong.** A configuration layout that keeps secrets outside the document root and outside version control, with the include path handled explicitly.
- **The truncation trap, explained.** Why a closing PHP tag in a comment ends the script, what the client receives as a result, and how to spot it quickly.
- **The two shapes of the fix.** Either remove the closing tag entirely, or restructure the file so the comment cannot terminate it.
- **Diagnosis cues.** The specific symptom pattern — every endpoint returning source fragments simultaneously — and why that pattern points here rather than at deployment.

## Requirements

PHP · a typical panel-hosted environment (Nginx or Apache)

## Installation

```bash
git clone https://github.com/<your-username>/php-secret-config-guard.git
```

Then place the folder into your agent skills directory:

| OS | Path |
|---|---|
| Windows | `C:\Users\<you>\.workbuddy\skills\php-secret-config-guard\` |
| macOS / Linux | `~/.workbuddy/skills/php-secret-config-guard/` |

There is no build step. Restart the session and it takes effect.

## How to use

Bring up the situation:

- "Where should I put the API keys in this PHP project?"
- "All my PHP endpoints are returning source code — what happened?"
- "Is it safe to put credentials in this file?"

The skill covers both the layout question and the debugging path.

## Files

```
└── SKILL.md
```

## Notes and limits

1. **The truncation trap fails open, not closed.** The file does not error out. It serves its own contents. Assume anything in that file was exposed.
2. **Fix the layout, not just the file.** Moving the closing tag solves one instance of the problem; keeping secrets outside the document root removes the class.
3. **Check what was already served.** If this happened in production, rotate the credentials in that file. Removing the offending line does not un-send what was already sent.

## License

MIT
