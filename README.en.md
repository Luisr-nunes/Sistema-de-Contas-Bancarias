# Bank Account Management System

<!-- badges -->
![C](https://img.shields.io/badge/C-A8B9CC?style=flat-square&logo=c&logoColor=white) ![Make](https://img.shields.io/badge/Make-427819?style=flat-square&logo=gnu&logoColor=white) ![GCC](https://img.shields.io/badge/GCC-A42E2B?style=flat-square&logo=gnu&logoColor=white)

[Ler em Português](README.md)

> Academic project written in **C11** with a layered modular architecture, binary file operations and an organization inspired by CRUD/enterprise principles.

---

## Contents

- [About](#about)
- [Features](#features)
- [Modular architecture](#modular-architecture)
- [Why a layered architecture?](#why-a-layered-architecture)
- [Directory structure](#directory-structure)
- [Modules](#modules)
- [Data model](#data-model)
- [Binary file operations](#binary-file-operations)
- [Dependency diagram](#dependency-diagram)
- [Building and running](#building-and-running)
- [Technical decisions](#technical-decisions)

---

## About

This system implements a **full CRUD of bank accounts** persisted in a binary file of fixed-size records. It was built as a hands-on exercise for:

- Binary file handling (`fopen`, `fread`, `fwrite`, `fseek`, `rewind`)
- Splitting code across multiple files by responsibility
- User input validation and structured error handling
- `typedef struct` with fixed-size fields for binary compatibility

What sets it apart from a monolithic implementation is the **clear separation of responsibilities between modules**, which makes each part testable, replaceable and understandable on its own.

---

## Features

| # | Operation | Description |
|---|-----------|-------------|
| 1 | **Register customer** | Inserts a new customer, reusing inactive slots before appending |
| 2 | **Look up customer** | Finds an account by number and shows its data |
| 3 | **Update balance** | Deposit, withdrawal or setting an exact amount, with validation |
| 4 | **Close account** | Logical deletion (sets `ativo = 0`); the slot becomes reusable |
| 5 | **List all** | Shows every active customer using `fseek` by position |
| 6 | **Rewind read** | Demonstrates an explicit `rewind()` and sequential re-read from byte 0 |
| 7 | **Exit** | Quits the program |

The program's menus and messages are in Portuguese.

---

## Modular architecture

The project uses a **three-layer architecture** inspired by enterprise systems and medium to large C applications:

```
┌─────────────────────────────────────────────────────┐
│                   PRESENTATION                      │
│              main.c  ·  menu.c                      │
│       (user interaction, output formatting)         │
├─────────────────────────────────────────────────────┤
│                    VALIDATION                       │
│                  validacao.c                        │
│   (safe stdin reading, input business rules)        │
├─────────────────────────────────────────────────────┤
│               DATA ACCESS LAYER (DAL)               │
│                     db.c                            │
│    (fopen · fread · fwrite · fseek · rewind)        │
└─────────────────────────────────────────────────────┘
              shared model:
                  cliente.h
```

Each layer **only knows the layer directly below it**, through the public interface declared in its `.h` file. No layer skips another.

---

## Why a layered architecture?

### The problem with a monolithic approach

A C program with everything in one file works for small examples but runs into structural problems as it grows:

- **Tight coupling:** changing how the file is opened can break the menu logic
- **No isolated testing:** testing validation means running the whole program
- **Hard maintenance:** finding where to fix a bug means reading hundreds of lines without clear context
- **No reuse:** the integer-reading function cannot easily be reused in another project

### The modular solution

Splitting into modules solves each of these:

**Fault isolation** — if `db.c` has a write bug, `menu.c` and `validacao.c` still compile and work. The error stays **contained in its own layer**.

**Replaceability** — `db.c` could be rewritten to use SQLite, PostgreSQL or any other backend without changing a single line of `menu.c`, which only sees the `DbStatus` enum and the `Cliente` struct.

**Testability** — `validacao.c` has no dependency on other project modules. It can be compiled and tested completely on its own, feeding inputs and checking outputs.

**Readability** — opening `db.c`, the reader knows to expect *only* file logic; opening `menu.c`, *only* presentation and orchestration. The context is implied by the file's location.

**Incremental builds** — the `Makefile` compiles each `.c` into its own `.o`. Changing only `menu.c` recompiles only `menu.o`.

### Enterprise analogy

In professional stacks (Java Spring, .NET, Node.js), this separation has formal names:

| Enterprise layer | Equivalent here |
|-------------------|----------------|
| Controller / View | `menu.c` |
| Service / Business Logic | `menu.c` (`op_*` functions) |
| Repository / DAO | `db.c` |
| DTO / Model | `cliente.h` |
| Utility / Helper | `validacao.c` |

The difference is that C has no classes or frameworks to enforce the separation; it is kept by **architectural discipline**: each `.c` includes only the headers it really needs, and internal functions are declared `static` to block outside access.

---

## Directory structure

```
banco/
│
├── Makefile                  # Build orchestration
│
├── include/                  # Public interfaces (contracts between modules)
│   ├── cliente.h             # Cliente struct + domain constants
│   ├── db.h                  # Data layer interface + DbStatus
│   ├── menu.h                # Presentation layer interface
│   └── validacao.h           # Validation and input interface
│
└── src/                      # Implementations
    ├── main.c                # Entry point
    ├── db.c                  # Data Access Layer
    ├── menu.c                # Controller + UI
    └── validacao.c           # Safe input
```

> **Note:** `contas.dat` (data file created at runtime) and `build/` (compiled objects) are listed in `.gitignore`, but `banco/contas.dat` and `banco/sistema.exe` were added before it and are still tracked.

---

## Modules

### `cliente.h` — Core data model

Defines `struct Cliente` and the domain constants. It is the only header included by every other module, and it includes no project header itself.

```c
typedef struct {
    int    numero;           // primary key
    char   nome[50];         // fixed size, required for binary I/O
    double saldo;
    int    ativo;            // CONTA_ATIVA | CONTA_INATIVA
} Cliente;
```

**Why `char nome[50]` instead of `char *nome`?**
Because `sizeof(Cliente)` must be a compile-time constant. With a pointer, `sizeof` would return the pointer size (8 bytes on 64-bit), not the name's, and `fseek(pos * sizeof(Cliente))` would silently break.

---

### `db.h` / `db.c` — Data Access Layer

**The only module that uses file functions.** No other `.c` file calls `fopen`, `fread`, `fwrite`, `fseek` or `rewind` directly.

It returns a `DbStatus` enum instead of exposing internal details:

```c
typedef enum {
    DB_OK            =  0,
    DB_ERR_ARQUIVO   = -1,
    DB_ERR_IO        = -2,
    DB_ERR_NAO_FOUND = -3,
    DB_ERR_DUPLICADO = -4,
    DB_ERR_SALDO     = -5
} DbStatus;
```

Helper functions (`db_abrir`, `offset_de`) are declared `static`, invisible outside `db.c`, the equivalent of `private` methods.

`db_listar` uses the **callback/iterator** pattern:

```c
typedef void (*db_iter_fn)(const Cliente *c, int pos, void *userdata);
int db_listar(db_iter_fn callback, void *userdata);
```

This lets `menu.c` decide *how* to display each record without `db.c` knowing anything about formatting.

---

### `validacao.h` / `validacao.c` — Input layer

Depends on no other project module. It centralizes all `stdin` reading so the program never hangs on invalid input:

- `val_ler_int()` — repeats until a valid integer is read
- `val_ler_double()` — repeats until a valid number is read
- `val_ler_string()` — uses `fgets`, strips `\n`, rejects empty strings
- `val_numero_conta()` / `val_saldo()` — validation business rules

---

### `menu.h` / `menu.c` — Presentation and controller

Orchestrates the CRUD operations by calling `db_*` and `val_*`. All output formatting lives here. It interprets `DbStatus` and shows friendly messages through `imprimir_status()`.

There is a single public function, `menu_executar()`; `main.c` needs to know nothing about how the menu works inside.

---

### `main.c` — Entry point

A few lines with a single responsibility: print the banner and call `menu_executar()`. No business logic.

---

## Data model

`contas.dat` is a binary file of **fixed-size records**. Each record takes exactly `sizeof(Cliente)` bytes (typically 72 bytes on 64-bit systems with default alignment).

```
Byte 0                                          Byte 71
┌──────────┬────────────────────────┬───────────┬──────────┐
│  numero  │         nome           │   saldo   │  ativo   │
│  4 bytes │       50 bytes         │  8 bytes  │  4 bytes │
└──────────┴────────────────────────┴───────────┴──────────┘
```

**Logical deletion:** closed accounts get `ativo = 0` but the slot stays in the file. On the next insert, the system scans for the first inactive slot and reuses it before appending.

---

## Binary file operations

| Function | Where | Purpose |
|--------|-------------|------------|
| `fopen("r+b")` | `db_abrir()` | Open for reading and writing without truncating |
| `fopen("w+b")` | `db_abrir()` (fallback) | Create the file on first run |
| `fseek(pos * sizeof(Cliente), SEEK_SET)` | `db_buscar`, `db_inserir`, `db_atualizar` | Direct access to a record by index |
| `fread(&c, sizeof(Cliente), 1, fp)` | Every read operation | Read the whole struct at once |
| `fwrite(&c, sizeof(Cliente), 1, fp)` | `db_inserir`, `db_atualizar` | Write or overwrite a record |
| `rewind(fp)` | `db_listar()` | Go back to byte 0 before a sequential scan |
| `ftell(fp)` | `db_total_registros()` | Count records from the file size |

---

## Dependency diagram

```
main.c
  └── menu.h
        │
        ▼
      menu.c ──────────────── cliente.h
        │         │
        ▼         ▼
       db.h    validacao.h
        │         │
        ▼         ▼
       db.c   validacao.c
        │
        ▼
    cliente.h
```

Ground rule: **no circular dependencies**. `db.c` does not know `menu.c`, `validacao.c` knows nobody and `cliente.h` includes no project header.

---

## Building and running

### Requirements

- GCC with C11 support (`gcc --version`)
- Make (`make --version`)
- Linux, macOS or Windows with MSYS2/MinGW

### Commands

```bash
git clone https://github.com/Luisr-nunes/Sistema-de-Contas-Bancarias.git
cd Sistema-de-Contas-Bancarias/banco

make           # build
make run       # build and run
make clean     # remove build artifacts and the data file
make rebuild   # full rebuild
```

The executable is written to `build/banco` (or `build/banco.exe` on Windows).

### `.gitignore`

```
build/
contas.dat
*.exe
*.o
```

---

## Technical decisions

**Why `static` on `db.c` internal functions?**
File-scope `static` hides the symbol outside its translation unit; it is C's `private`. Without it, `db_abrir()` would be reachable from any `.c` file, breaking the data layer's encapsulation.

**Why reuse inactive slots instead of compacting the file?**
Compacting would mean rewriting every record after the deleted one, an O(n) operation that could corrupt the file if interrupted. Slot reuse keeps the lookup O(n) but the write O(1), and is the pattern real databases use ("tombstone deletion").

**Why `double` for the balance instead of `int` (cents)?**
For teaching purposes, `double` is more direct. In real financial systems, `double` has floating-point precision issues (`0.1 + 0.2 ≠ 0.3` in IEEE 754). The correct approach is to store cents in a `long int` and divide by 100 for display.

**Why a `db_iter_fn` callback in `db_listar`?**
The callback iterator decouples *what to fetch* (`db.c`'s job) from *what to do with each result* (`menu.c`'s job). Exporting to CSV tomorrow would only need a different callback, with no change to `db.c`.

---

> Academic project for the Imperative and Functional Programming course — CESAR School, Recife, Brazil.

## Author

Luís Nunes — [GitHub](https://github.com/Luisr-nunes) · [LinkedIn](https://www.linkedin.com/in/luis-nunes-dev/)
