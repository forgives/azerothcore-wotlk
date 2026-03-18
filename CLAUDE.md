# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AzerothCore is an open-source game server application and framework designed for hosting massively multiplayer online role-playing games (MMORPGs). It is based on World of Warcraft WotLK (3.3.5a) and seeks to recreate the gameplay experience of the original game.

**Key project goals:**
- **Stability** - All changes pass CI before merging
- **Blizzlike content** - High standard for fixes to match official gameplay
- **Customization** - Easy to customize using modules
- **Community driven** - Active developer and user community

The original code is based on MaNGOS, TrinityCore, and SunwellCore.

## Build System

### Quick Build Commands

```bash
# Interactive menu system - recommended for most operations
./acore.sh compiler              # Enter compiler menu
./acore.sh compiler build        # Configure and compile
./acore.sh compiler all          # Clean, configure and compile
./acore.sh compiler clean        # Clean build files

# Direct CMake (manual build)
mkdir build && cd build
cmake .. -DSCRIPTS=static -DMODULES=static
make -j$(nproc)
```

### Key Build Options (CMake)

- `SCRIPTS`: Script linking type (`none`, `static`, `dynamic`, `minimal-static`, `minimal-dynamic`)
- `MODULES`: Module linking type (`none`, `static`, `dynamic`)
- `BUILD_APPS`: Which applications to build (`all`, `auth-only`, `world-only`, `none`)
- `BUILD_TESTING`: Enable unit tests
- `LUA_VERSION`: Lua version (`luajit`, `lua52`, `lua53`, `lua54`)
- `WITH_WARNINGS`: Show all compiler warnings
- `WITH_COREDEBUG`: Include additional debug code

### Configuration

Edit build settings in `conf/config.cmake` or set environment variables. The build system is CMake-based (requires 3.16-3.22).

## Testing

```bash
# Run unit tests
./acore.sh test core

# Run bash tests
./acore.sh test bash

# Run tests manually from build directory
cd build && ctest --verbose

# Generate code coverage report (requires BUILD_TESTING=ON)
cd build && make coverage
# Coverage report will be in coverage-report/ directory
```

Unit tests use Google Test framework and are located in `src/test/`. Test directory structure mirrors the source code structure. Code coverage reports use lcov/genhtml.

**Running specific tests:**
```bash
# List all tests
cd build && ctest --show-only

# Run specific test by name
cd build && ctest --verbose -R TestName
```

## Code Style and Linting

```bash
# Check C++ code style
python apps/codestyle/codestyle-cpp.py

# Check SQL code style
python apps/codestyle/codestyle-sql.py
```

The project uses custom code style checkers for C++ and SQL. These are enforced via GitHub Actions CI.

## Logging System

AzerothCore uses a log4j-like logging system with loggers and appenders.

**Log levels (from highest to lowest):** `FATAL`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`, `DISABLED`

**Appender types:**
- `1` - Console
- `2` - File
- `3` - Database

**Configuration:**
Logging is configured in the server config files via `Logger.*` and `Appender.*` entries.

**Example usage in code:**
```cpp
LOG_INFO("entities.player", "Player {} logged in", playerName);
LOG_ERROR("guild", "Guild {} creation failed", guildId);
```

See `doc/Logging.md` for comprehensive documentation.

## Architecture

### Core Applications (`apps/`)

- `authserver` - Authentication and account management
- `worldserver` - Main game server handling all gameplay logic
- Extraction tools (`dbcextract`, `mapextractor`, `vmap4_extractor`) - Extract game data from client

### Source Structure (`src/`)

- `src/server/game/` - Core gameplay logic
  - `Entities/` - Game objects (Player, Creature, GameObject, Item, Unit, Pet, Vehicle, etc.)
  - `Handlers/` - Packet handlers for client-server communication
  - `Maps/` - Map, grid, and instance management
  - `Spells/` - Spell system, auras, and effects
  - `AI/` - AI systems (CreatureAI, PlayerAI, MovementAI)
  - `Scripting/` - Script system interfaces and managers
  - `Combat/` - Combat, threat, and damage systems
  - `Battlegrounds/`, `Arenas/` - PvP systems
  - `Groups/`, `Guilds/` - Social systems
- `src/server/database/` - Database connectivity, migrations, and updaters
- `src/server/shared/` - Shared utilities (Network, Configuration, DataStores, Packets)
- `src/tools/` - Data extraction and generation tools
- `src/common/` - Common code shared across all components

### Module System (`modules/`)

AzerothCore uses a dynamic module loading system:

- Modules can be linked statically (into worldserver) or dynamically (loaded at runtime)
- Each module is a subdirectory in `modules/` with its own `CMakeLists.txt`
- Build configuration: `-DMODULES=static` or `-DMODULES=dynamic`
- Module loading is handled by `ModulesLoader.cpp` (generated from template)
- Modules can include config files in `modules/<name>/conf/*.conf.dist`

**Creating a new module:**
```bash
cd modules
./create_module.sh  # Interactive script to scaffold a new module
```

Or manually clone from https://github.com/azerothcore/skeleton-module/

### Database Architecture

- Three databases: `auth`, `characters`, `world`
- SQL updates go in `data/sql/updates/pending_db_*/`
- Migration system handles schema updates
- DBC files contain client game data

**Working with SQL:**
- Place pending SQL updates in the appropriate `pending_db_*` directory
- Follow the SQL code style guidelines
- Reference wowhead, wowgaming, or similar sources for data validation
- See http://www.azerothcore.org/wiki/Dealing-with-SQL-files for detailed guidelines

### Scripting

- C++ scripts for core functionality
- Lua scripting via `mod-eluna` module (LuaJIT-based)
- SmartAI (SAI) for database-driven NPC scripting
- Script loaders are generated at build time

## Commit Message Format

Follow the template in `.git_commit_template.txt`:

```
### TITLE
## Type(Scope/Subscope): Short description (max 50 chars)

### DESCRIPTION
## Explain what and why (max 72 chars per line)

## Provide links to issues, commits, or references
```

Types: `feat`, `fix`, `refactor`, `style`, `docs`, `test`, `chore`
Scopes: `CORE`, `DB`, and optional subscopes

## Configuration Files

- Server configs: `conf/worldserver.conf.dist`, `conf/authserver.conf.dist`
- Build config: `conf/config.cmake`, `conf/dist/config.sh`
- Config loader supports severity policies (see `doc/ConfigPolicy.md`)

## Important Development Notes

- Default build type is `RelWithDebInfo` (Release with Debug Info)
- Precompiled headers (PCH) are enabled by default for faster compilation
- ccache can be enabled for faster rebuilds
- The `acore.sh` script provides a unified interface for all common operations
- Module-specific configs are automatically merged during build
- macOS builds require special paths for MySQL, OpenSSL, and Readline (handled by compiler script)

## Key Conventions

- Classes use PascalCase (`Player`, `GameObject`)
- Member variables use camelCase
- Smart pointers and RAII patterns throughout
- Extensive use of templates and inheritance
- Packet handlers follow naming pattern: `Handle*Opcode`

## Useful acore.sh Commands

```bash
./acore.sh init                  # First-time setup
./acore.sh install-deps          # Install OS dependencies
./acore.sh module list           # List available modules
./acore.sh module install <name> # Install a module
./acore.sh config                # Configuration manager
./acore.sh run-worldserver       # Start worldserver with restarter
./acore.sh run-authserver        # Start authserver with restarter
```

## Important Resources

- **Wiki:** http://www.azerothcore.org/wiki
- **Doxygen Documentation:** https://www.azerothcore.org/pages/doxygen/index.html
- **Module Catalogue:** http://www.azerothcore.org/catalogue.html
- **Discord:** https://discord.gg/gkt4y2x
- **StackOverflow:** https://stackoverflow.com/questions/tagged/azerothcore
- **Forum:** https://github.com/azerothcore/azerothcore-wotlk/discussions/
- **Issue Tracker:** https://github.com/azerothcore/azerothcore-wotlk/issues
- **Website:** http://www.azerothcore.org/

**Game data references:**
- https://wowgaming.altervista.org/aowow/ (WotLK database)
- https://www.wowhead.com/wotlk/
- https://wowpedia.fandom.com/
