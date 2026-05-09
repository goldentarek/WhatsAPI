# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running Tests

Tests are plain PHP scripts run from the `tests/` directory — no test framework is installed:

```bash
cd tests && php WhatAppEventTest.php
```

There is no build step. The library is pure PHP with no Composer dependencies.

## Code Style

PSR-2. All methods are camelCase. Use [PHP Coding Standards Fixer](http://cs.sensiolabs.org/) to auto-fix:

```bash
php-cs-fixer fix src/php/
```

## Architecture

### Entry Point: `WhatsProt`

`src/php/whatsprot.class.php` is the public API. Instantiate with phone number, identity token, and nickname:

```php
$w = new WhatsProt($phone, $identity, $nickname, $debug);
$w->connect();
$w->loginWithPassword($password);
```

After login, use `sendMessage()`, `sendMessageImage()`, `sendLocation()`, etc. The message loop calls `pollMessages()` in a tight loop to receive inbound data.

### Protocol Layer

`protocol.class.php` implements FunXMPP (WhatsApp's binary XMPP variant). The key data structure is `ProtocolNode` — a tree node with a tag, attribute hash, child nodes, and raw data. `BinTreeNodeWriter` serializes these to binary using a token dictionary (`func.php::getDictionary()`), and `BinTreeNodeReader` deserializes them. All hex literals scattered through the code are WhatsApp's proprietary token dictionary entries.

### Encryption

All socket traffic is RC4-encrypted after the initial handshake. `WhatsProt` holds `$inputKey` and `$outputKey` as `rc4` instances (from `rc4.php`). The stream cipher state is maintained across successive `cipher()` calls — do not reset mid-session.

### Event System (4-class hierarchy)

```
WhatsAppEventListener (interface, ~40 methods)
  ├── WhatsAppEventListenerBase      — empty no-op implementations; extend this to handle only specific events
  └── WhatsAppEventListenerProxy     — abstract; routes ALL events to a single handleEvent($name, $args) method
        └── WhatsAppEventListenerLegacyAdapter — supports the old bind($eventName, $callback) API
```

`WhatsAppEvent` holds a static array of registered listeners and provides `addEventListener()` (new API) and `bind()` (legacy). Every `WhatsProt` method fires typed events (e.g. `fireGetMessage()`) which dispatch to all registered listeners.

**To handle events**, extend `WhatsAppEventListenerBase` and override only the methods you need, then call `$w->eventManager()->addEventListener(new MyListener())`.

### Authentication & Identity

`token.php::generateRequestToken()` creates the HMAC token using the hardcoded WhatsApp Android signing certificate. The `$identity` parameter to `WhatsProt` is a device fingerprint: if not already a valid hash, `WhatsProt` SHA1-hashes it and stores state in `nextChallenge.dat` and `magic.dat` in `src/php/`. These files must persist between sessions.

### JID Format

- Users: `{phone}@s.whatsapp.net`
- Groups: `{creator_phone}-{creation_timestamp}@g.us`

Phone numbers never include `+` or `00` prefixes.

### Media

`mediauploader.php` HTTP-uploads files to `https://mms.whatsapp.net/...` before sending. The resulting URL is embedded in the message node. Local media files go in `src/php/media/`; profile pictures in `src/php/pictures/`.

### File Loading

No autoloader. All files use manual `require_once` with paths relative to `src/php/`. Test files load via `../src/php/whatsprot.class.php`.

## Workflow

### Planning

Enter plan mode for any non-trivial task (3+ steps or architectural decisions). Write the plan to `tasks/todo.md` with checkable items before writing any code. If something goes sideways mid-task, stop and re-plan immediately. Use plan mode for verification steps, not just building.

### Subagents

Use subagents liberally to keep the main context window clean. Offload research, exploration, and parallel analysis to subagents. One task per subagent for focused execution. For complex problems, throw more compute at it via subagents.

### Self-Improvement Loop

After any correction from the user, update `tasks/lessons.md` with the pattern: what the mistake was, why it happened, and the rule to prevent it going forward. Review `tasks/lessons.md` at the start of each session.

### Verification

Never mark a task complete without proving it works. Run tests, check logs, demonstrate correctness. Ask: "Would a staff engineer approve this?"

### Elegance

For non-trivial changes, pause and ask "is there a more elegant way?" If a fix feels hacky, implement the elegant solution instead. Skip this for simple, obvious fixes.

### Bug Fixing

When given a bug report, fix it — don't ask for hand-holding. Point at logs, errors, failing tests, then resolve them.

### Task Management Steps

1. Write plan to `tasks/todo.md` with checkable items
2. Check in before starting implementation
3. Mark items complete as you go
4. Give a high-level summary at each step
5. Add a review section to `tasks/todo.md` when done
6. Update `tasks/lessons.md` after any user correction

### Core Principles

- **Simplicity first:** Make every change as simple as possible. Impact minimal code.
- **No laziness:** Find root causes. No temporary fixes.
- **Minimal impact:** Only touch what's necessary. No side effects.
