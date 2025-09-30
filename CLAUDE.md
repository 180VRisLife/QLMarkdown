# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

QLMarkdown is a macOS Quick Look extension for previewing Markdown files. It includes a settings application, Quick Look extension, CLI tool, XPC helper, and Shortcut extension.

## Build Commands

### Building the Project
```bash
# Open the project in Xcode
open QLMarkdown.xcodeproj

# Build from command line (requires xcodebuild)
xcodebuild -project QLMarkdown.xcodeproj -scheme QLMarkdown -configuration Release
```

### Building Dependencies

Dependencies must be built before building the main project:

```bash
# Install required build tools
brew install autoconf cmake go

# Build cmark-gfm (requires cmake)
cd cmark-gfm
# Follow build instructions in submodule

# Build highlight wrapper and dependencies
cd highlight-wrapper
make

# Build PCRE2 (requires autoconf)
cd dependencies
make -f MakefilePCRE

# Build JPCRE2
make -f MakefileJPCRE
```

## Architecture

### Main Components

- **QLMarkdown** (`QLMarkdown/`): Main macOS application for configuring Quick Look extension settings. Provides UI for themes, extensions, and options.
- **QLExtension** (`QLExtension/`): The actual Quick Look preview extension that renders Markdown files. Communicates with XPC helper for rendering.
- **QLMarkdownXPCHelper** (`QLMarkdownXPCHelper/`): XPC service that handles markdown processing isolated from main app for security/stability.
- **qlmarkdown_cli** (`qlmarkdown_cli/`): Command-line tool for batch conversion. Located at `QLMarkdown.app/Contents/Resources/qlmarkdown_cli`.
- **Shortcut Extension** (`Shortcut Extension/`): Shortcuts app integration for markdown conversion.

### C/C++ Libraries

- **cmark-gfm**: GitHub-flavored Markdown parser (submodule)
- **cmark-extra** (`cmark-extra/`): Custom extensions to cmark including:
  - Emoji support
  - Heads anchors
  - Highlight
  - Inline local images
  - Math expressions
  - Subscript/superscript
  - Syntax highlighting
  - YAML header parsing
- **highlight-wrapper** (`highlight-wrapper/`): Wrapper around the Highlight library for syntax highlighting. Includes:
  - `libmagic` for simple language detection
  - Enry (Go-based) for accurate language detection (similar to GitHub Linguist)
  - Lua interpreter for Highlight scripts
  - Boost libraries
- **dependencies** (`dependencies/`): PCRE2 and JPCRE2 for regex in heads extension

### Settings Architecture

The `Settings` class (`QLMarkdown/Settings.swift`) is shared across all components with platform-specific extensions:
- `Settings+NoXPC.swift`: Direct rendering without XPC (CLI, Shortcuts)
- `Settings+XPC.swift`: XPC-based rendering (main app, Quick Look extension)
- `Settings+render.swift`: Core rendering logic using cmark-gfm
- `Settings+ext.swift`: Various utility extensions

Settings are persisted to shared preferences and synchronized via `NSNotification.Name.QLMarkdownSettingsUpdated`.

### Rendering Flow

1. Quick Look extension receives request → loads Settings from shared preferences
2. Settings+XPC calls QLMarkdownXPCHelper via XPC
3. XPC helper uses cmark-gfm + custom extensions to parse Markdown
4. Syntax highlighting handled by highlight-wrapper library
5. HTML output injected with CSS theme and inline images
6. WebView displays final HTML in Quick Look window

## Key File Types Supported

- Markdown: `.md`, `.markdown`
- RMarkdown: `.rmd` (without R code evaluation)
- Quarto: `.qmd`
- MDX: `.mdx` (without JSX rendering)
- Cursor Rulers: `.mdc`
- API Blueprint: `.apib`
- Textbundle: `.textbundle` packages

## Important UTI Handling

The extension registers for multiple Markdown UTIs (see README.md for complete list). Check UTI assignment with:
```bash
touch /tmp/test.md && mdls -name kMDItemContentType /tmp/test.md && rm /tmp/test.md
```

## CLI Tool Usage

```bash
# Create symbolic link for easy access
ln -s /Applications/QLMarkdown.app/Contents/Resources/qlmarkdown_cli /usr/local/bin/qlmarkdown_cli

# Convert single file to stdout
qlmarkdown_cli file.md

# Convert to specific output
qlmarkdown_cli -o output.html file.md

# Convert multiple files to directory
qlmarkdown_cli -o output_dir/ file1.md file2.md
```

CLI uses same settings as Quick Look extension but allows overrides via flags.

## Swift Package Dependencies

Managed by Swift Package Manager:
- **Sparkle**: Auto-update framework
- **Yams**: YAML parsing for headers
- **SwiftSoup**: HTML parsing for inline images

Reset package cache if issues occur: File → Packages → Reset Package Caches

## Security Considerations

- The app has read-only entitlements for the entire filesystem to preview local images
- XPC helper provides isolation between main app and markdown processing
- `com.apple.security.temporary-exception.mach-lookup.global-name` entitlement addresses Big Sur WebKit bug
- Unsafe HTML is disabled by default; enable only if needed for SVG/raw HTML
- Inline images only process `file://` URLs or relative paths to local files