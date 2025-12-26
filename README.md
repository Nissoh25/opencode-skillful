# OpenCode Skills Plugin

An interpretation of the [Anthropic Agent Skills Specification](https://github.com/anthropics/skills) for OpenCode, providing lazy-loaded skill discovery and injection.

Differenator is :

- Conversationally the agent uses `skill_find words, words words` to discover skills
- The agent uses `skill_use fully_resolved_skill_name` and,
- The agent can use `skill_resource skill_relative/resource/path` to read reference material

## Installation

Create or edit your OpenCode configuration file (typically `~/.config/opencode/config.json`):

```json
{
  "plugins": ["@zenobius/opencode-skillful"]
}
```

## Usage

### Example 1: Finding and Loading Skills

```
I need to write a commit message. Can you find any relevant skills and load them?

1. Use skill_find to search for "commit" or "git" related skills
2. Load the most relevant skill using skill_use
3. Apply the loaded skill guidance to write my commit message
```

**Demonstrates:**

- Searching for skills by keyword
- Loading skills into the chat context
- Applying skill guidance to tasks

### Example 2: Browsing Available Skills

```
What skills are available? Show me everything under the "experts" category.

1. List all skills with skill_find "*"
2. Filter to a specific path with skill_find "experts"
3. Load a specific expert skill for deep guidance
```

**Demonstrates:**

- Listing all available skills
- Path prefix filtering (e.g., "experts", "superpowers/writing")
- Hierarchical skill organization

### Example 3: Advanced Search with Exclusions

```
Find testing-related skills but exclude anything about performance testing.

skill_find "testing -performance"
```

**Demonstrates:**

- Natural language query syntax
- Negation with `-term`
- AND logic for multiple terms

## Features

- :mag: Discover SKILL.md files from multiple locations
- :zap: Lazy loading - skills only inject when explicitly requested
- :file_folder: Path prefix matching for organized skill browsing
- :abc: Natural query syntax with negation and quoted phrases
- :label: Skill ranking by relevance (name matches weighted higher)
- :recycle: Silent message insertion (noReply pattern)

## Skill Discovery Paths

Skills are discovered from these locations (in priority order, last wins on duplicates):

1. `~/.opencode/skills/` - User global skills (lowest priority)
2. `~/.config/opencode/skills/` - Standard XDG config location
3. `.opencode/skills/` - Project-local skills (highest priority)

## Usage in OpenCode

### Finding Skills

```
# List all available skills
skill_find query="*"

# Search by keyword
skill_find query="git commit"

# Path prefix matching
skill_find query="experts/data-ai"

# Exclude terms
skill_find query="testing -performance"
```

### Loading Skills

```
# Load a single skill
skill_use skill_names=["writing-git-commits"]

# Load multiple skills
skill_use skill_names=["writing-git-commits", "code-review"]

# Load by fully-qualified name
skill_use skill_names=["experts/writing-git-commits"]
```

### Reading Skill Resources

```
# Read a reference document
skill_resource skill_name="writing-git-commits" relative_path="references/style-guide.md"

# Read a template
skill_resource skill_name="brand-guidelines" relative_path="assets/logo-usage.html"
```

### Executing Skill Scripts

```
# Execute a script without arguments
skill_exec skill_name="build-utils" relative_path="scripts/generate-changelog.sh"

# Execute a script with arguments
skill_exec skill_name="code-generator" relative_path="scripts/gen.py" args=["--format", "json", "--output", "schema.json"]
```

## Plugin Tools

The plugin provides four core tools implemented in `src/tools/`:

### `skill_find` (SkillFinder.ts)

Search for skills using natural query syntax with intelligent ranking by relevance.

**Parameters:**

- `query`: Search string supporting:
  - `*` or empty: List all skills
  - Path prefixes: `experts`, `superpowers/writing` (prefix matching)
  - Keywords: `git commit` (multiple terms use AND logic)
  - Negation: `-term` (exclude results)
  - Quoted phrases: `"exact match"` (phrase matching)

**Returns:**

- List of matching skills with:
  - `skill_name`: Fully-qualified tool name (FQDN)
  - `skill_shortname`: Short skill identifier
  - `description`: Human-readable description
- Debug information (when enabled): discovered count, parsed results, rejections, duplicates, errors

**Example Response:**

```xml
<SkillSearchResults query="git commit">
  <Skills>
    <Skill skill_name="experts/writing-git-commits" skill_shortname="writing-git-commits">
      Guidelines for writing effective git commit messages
    </Skill>
  </Skills>
  <Summary>
    <Total>42</Total>
    <Matches>1</Matches>
    <Feedback>Found 1 skill matching your query</Feedback>
  </Summary>
</SkillSearchResults>
```

### `skill_use` (SkillUser.ts)

Load one or more skills into the chat context with full resource metadata.

**Parameters:**

- `skill_names`: Array of skill names to load (by toolName/FQDN or short name)

**Features:**

- Injects skill metadata and content including:
  - Skill name and description
  - Resource inventory:
    - `references`: Documentation and reference files
    - `assets`: Binary files, templates, images
    - `scripts`: Executable scripts with paths and MIME types
  - Full skill content formatted as Markdown for easy reading
  - Base directory context for relative path resolution

**Behavior:** Silently injects skills as user messages (persists in conversation history)

**Example Response:**

```json
{ "loaded": ["experts/writing-git-commits"], "notFound": [] }
```

### `skill_resource` (SkillResourceReader.ts)

Read a specific resource file from a skill's directory and inject silently into the chat.

**Parameters:**

- `skill_name`: The skill containing the resource (by toolName/FQDN or short name)
- `relative_path`: Path to the resource relative to the skill directory

**Returns:**

- MIME type of the resource
- Success confirmation with skill name, resource path, and type

**Behavior:**

- Silently injects resource content without triggering AI response (noReply pattern)
- Resolves relative paths within skill directory
- Supports any text or binary file type

**Example Response:**

```
Load Skill Resource

  skill: experts/writing-git-commits
  resource: templates/commit-template.md
  type: text/markdown
```

### `skill_exec` (SkillScriptExec.ts)

Execute scripts from skill resources with optional arguments.

**Parameters:**

- `skill_name`: The skill containing the script (by toolName/FQDN or short name)
- `relative_path`: Path to the script file relative to the skill directory
- `args`: Optional array of string arguments to pass to the script

**Returns:**

- Exit code of the script execution
- Standard output (stdout)
- Standard error (stderr)
- Formatted text representation

**Behavior:**

- Locates and executes scripts within skill directories
- Passes arguments to the script
- Silently injects execution results
- Includes proper error handling and reporting

**Example Response:**

```
Executed script from skill "build-utils": scripts/generate-changelog.sh

Exit Code: 0
STDOUT: Changelog generated successfully
STDERR: (none)
```

## Architecture

The plugin consists of two main layers:

### Services Layer (`src/services/`)

Core business logic for skill management:

- **SkillProvider**: Main interface for accessing the skill registry and searcher
- **SkillRegistry**: Manages skill storage, lookup, and lifecycle
- **SkillSearcher**: Implements search parsing and matching logic
  - Supports natural language queries (Gmail-style syntax)
  - Handles negation, quoted phrases, and path prefix matching
  - Scores results by relevance (name matches weighted higher)
- **SkillFs**: Filesystem abstraction for skill discovery
- **SkillResourceResolver**: Resolves and reads resource files from skill directories
- **ScriptResourceExecutor**: Executes scripts from skill resources
- **OpenCodeChat**: Injects skill content and resources into chat context

### Tools Layer (`src/tools/`)

Four tool implementations that expose plugin functionality:

- **SkillFinder.ts**: `skill_find` - Search and discover skills
- **SkillUser.ts**: `skill_use` - Load skills into context
- **SkillResourceReader.ts**: `skill_resource` - Read skill resource files
- **SkillScriptExec.ts**: `skill_exec` - Execute skill scripts

Each tool:

1. Validates input parameters
2. Delegates to service layer logic
3. Formats results as XML/JSON
4. Silently injects skill content via OpenCodeChat
5. Returns human-readable feedback to the agent

## Creating Skills

Skills follow the [Anthropic Agent Skills Specification](https://github.com/anthropics/skills). Each skill is a directory containing a `SKILL.md` file:

```
skills/
  my-skill/
    SKILL.md
    resources/
      template.md
```

### Skill Structure

```
my-skill/
  SKILL.md                 # Required: Skill metadata and instructions
  references/              # Optional: Documentation and guides
    guide.md
    examples.md
  assets/                  # Optional: Binary files and templates
    template.html
    logo.png
  scripts/                 # Optional: Executable scripts
    setup.sh
    generate.py
```

### SKILL.md Format

```yaml
---
name: my-skill
description: A brief description of what this skill does (min 20 chars)
license: MIT
allowed-tools:
  - bash
  - read
metadata:
  author: Your Name
---
# My Skill

Instructions for the AI agent when this skill is loaded...
```

**Requirements:**

- `name` must match the directory name (lowercase, alphanumeric, hyphens only)
- `description` must be at least 20 characters
- Directory name and frontmatter `name` must match exactly

### Resource Types

Skill resources are automatically discovered and categorized:

- **References** (`references/` directory): Documentation, guides, and reference materials
  - Typically Markdown, plaintext, or HTML files
  - Accessed via `skill_resource` for reading documentation
- **Assets** (`assets/` directory): Templates, templates, images, and binary files
  - Can include HTML templates, images, configuration files
  - Useful for providing templates and examples to the AI
- **Scripts** (`scripts/` directory): Executable scripts that perform actions
  - Shell scripts (.sh), Python scripts (.py), or other executables
  - Executed via `skill_exec` with optional arguments
  - Useful for automation, code generation, or complex operations

## Considerations

- Skills are discovered at plugin initialization (requires restart to reload)
- Duplicate skill names are logged and skipped (last path wins)
- Tool restrictions in `allowed-tools` are informational (enforced at agent level)
- Skill content is injected as user messages (persists in conversation)
- Base directory context is provided for relative path resolution in skills

## Contributing

Contributions are welcome! Please file issues or submit pull requests on the GitHub repository.

## License

MIT License. See the [LICENSE](LICENSE) file for details.
