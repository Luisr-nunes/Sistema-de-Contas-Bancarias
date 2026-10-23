# Sistema de Manutenção de Contas Bancárias

<!-- badges -->
![C](https://img.shields.io/badge/C-A8B9CC?style=flat-square&logo=c&logoColor=white) ![Make](https://img.shields.io/badge/Make-427819?style=flat-square&logo=gnu&logoColor=white) ![GCC](https://img.shields.io/badge/GCC-A42E2B?style=flat-square&logo=gnu&logoColor=white)

[Read in English](README.en.md)

> Projeto acadêmico desenvolvido em **C11** com arquitetura modular em camadas, operações de arquivo binário e padrão de organização inspirado em princípios CRUD/enterprise.

---

## Sumário

- [Sobre o Projeto](#sobre-o-projeto)
- [Funcionalidades](#funcionalidades)
- [Arquitetura Modular](#arquitetura-modular)
- [Por que Arquitetura em Camadas?](#por-que-arquitetura-em-camadas)
- [Estrutura de Diretórios](#estrutura-de-diretórios)
- [Descrição dos Módulos](#descrição-dos-módulos)
- [Modelo de Dados](#modelo-de-dados)
- [Operações de Arquivo Binário](#operações-de-arquivo-binário)
- [Diagrama de Dependências](#diagrama-de-dependências)
- [Como Compilar e Executar](#como-compilar-e-executar)
- [Decisões Técnicas](#decisões-técnicas)

---

## Sobre o Projeto

Este sistema implementa um **CRUD completo de contas bancárias** persistido em arquivo binário de registros de tamanho fixo. O projeto foi desenvolvido como exercício prático das seguintes disciplinas de C:

- Manipulação de arquivos binários (`fopen`, `fread`, `fwrite`, `fseek`, `rewind`)
- Organização de código em múltiplos arquivos com separação por responsabilidade
- Validação de entrada do usuário e tratamento de erros estruturado
- Uso de `typedef struct` com campos de tamanho fixo para compatibilidade binária

O diferencial deste projeto em relação a uma implementação monolítica está na **separação clara de responsabilidades entre módulos**, tornando cada parte testável, substituível e compreensível de forma independente.

---

## Funcionalidades

| # | Operação | Descrição |
|---|----------|-----------|
| 1 | **Cadastrar cliente** | Insere novo cliente; reutiliza slots inativos antes de fazer append |
| 2 | **Consultar cliente** | Busca por número de conta e exibe os dados completos |
| 3 | **Atualizar saldo** | Depósito, saque ou definição de valor exato com validação |
| 4 | **Encerrar conta** | Exclusão lógica (marca `ativo = 0`); slot fica disponível para reuso |
| 5 | **Listar todos** | Exibe todos os clientes ativos com `fseek` por posição |
| 6 | **Restaurar leitura** | Demonstra `rewind()` explícito e releitura sequencial desde o byte 0 |
| 7 | **Encerrar** | Sai do programa |

---

## Arquitetura Modular

O projeto adota uma **arquitetura em três camadas** inspirada no padrão utilizado em sistemas enterprise e aplicações C de médio/grande porte:

```
┌─────────────────────────────────────────────────────┐
│                   APRESENTAÇÃO                      │
│              main.c  ·  menu.c                      │
│   (interação com o usuário, formatação de output)   │
├─────────────────────────────────────────────────────┤
│                    VALIDAÇÃO                        │
│                  validacao.c                        │
│  (leitura segura de stdin, regras de negócio input) │
├─────────────────────────────────────────────────────┤
│               ACESSO A DADOS (DAL)                  │
│                     db.c                            │
│    (fopen · fread · fwrite · fseek · rewind)        │
└─────────────────────────────────────────────────────┘
              modelo compartilhado:
                  cliente.h
```

Cada camada **só conhece a camada imediatamente abaixo** via interface pública declarada nos arquivos `.h`. Nenhuma camada "pula" outra.

---

## Por que Arquitetura em Camadas?

### O problema da abordagem monolítica

Um programa C com tudo em um único arquivo funciona bem em exemplos pequenos, mas apresenta problemas estruturais conforme cresce:

- **Acoplamento total:** uma mudança em como o arquivo é aberto pode quebrar a lógica do menu
- **Impossibilidade de teste isolado:** para testar a validação, é preciso rodar o programa inteiro
- **Manutenção difícil:** encontrar onde corrigir um bug exige ler centenas de linhas sem contexto claro
- **Reuso zero:** a função de leitura de inteiro não pode ser aproveitada em outro projeto facilmente

### A solução modular

A separação em módulos resolve cada um desses problemas:

**Isolamento de falhas** — se `db.c` tiver um bug de escrita, `menu.c` e `validacao.c` continuam compilando e funcionando corretamente. O erro fica **contido na sua camada de origem**.

**Substituibilidade** — a camada `db.c` pode ser reescrita para usar SQLite, PostgreSQL ou qualquer outro backend sem que `menu.c` precise mudar uma única linha. Ela só enxerga o enum `DbStatus` e a struct `Cliente`.

**Testabilidade** — `validacao.c` não tem nenhuma dependência de outros módulos do projeto. Pode ser compilado e testado de forma completamente independente, passando entradas e verificando saídas.

**Legibilidade** — ao abrir `db.c`, o leitor sabe que vai encontrar *apenas* lógica de arquivo. Ao abrir `menu.c`, vai encontrar *apenas* lógica de apresentação e orquestração. O contexto está implícito na localização do arquivo.

**Compilação incremental** — o `Makefile` compila cada `.c` em um `.o` separado. Se você alterar apenas `menu.c`, somente `menu.o` é recompilado. Em projetos grandes isso reduz o tempo de build de minutos para segundos.

### Analogia com sistemas enterprise

Em sistemas profissionais (Java Spring, .NET, Node.js), essa separação tem nomes formais:

| Camada enterprise | Equivalente neste projeto |
|-------------------|--------------------------|
| Controller / View | `menu.c` |
| Service / Business Logic | `menu.c` (funções `op_*`) |
| Repository / DAO | `db.c` |
| DTO / Model | `cliente.h` |
| Utility / Helper | `validacao.c` |

A diferença é que em C não há classes ou frameworks para impor a separação — ela é mantida por **disciplina arquitetural**: cada `.c` inclui apenas os `.h` de que realmente precisa, e funções internas são declaradas `static` para impedir acesso externo.

---

## Estrutura de Diretórios

```
banco/
│
├── Makefile                  # Orquestração de build
│
├── include/                  # Interfaces públicas (contratos entre módulos)
│   ├── cliente.h             # Struct Cliente + constantes de domínio
│   ├── db.h                  # Interface da camada de dados + DbStatus
│   ├── menu.h                # Interface da camada de apresentação
│   └── validacao.h           # Interface de validação e input
│
└── src/                      # Implementações
    ├── main.c                # Entry point
    ├── db.c                  # Data Access Layer
    ├── menu.c                # Controller + UI
    └── validacao.c           # Input seguro
```

> **Nota:** `contas.dat` (arquivo de dados gerado em runtime) e `build/` (objetos compilados) estão no `.gitignore`, mas `banco/contas.dat` e `banco/sistema.exe` foram adicionados antes dele e continuam versionados.

---

## Descrição dos Módulos

### `cliente.h` — Modelo de Dados Central

Define a `struct Cliente` e as constantes de domínio. É o único header incluído por todos os outros módulos, e ele mesmo não inclui nenhum header do projeto.

```c
typedef struct {
    int    numero;           // chave primária
    char   nome[50];         // tamanho fixo — obrigatório para I/O binário
    double saldo;
    int    ativo;            // CONTA_ATIVA | CONTA_INATIVA
} Cliente;
```

**Por que `char nome[50]` e não `char *nome`?**
Porque `sizeof(Cliente)` precisa ser constante em tempo de compilação. Com um ponteiro, `sizeof` retornaria o tamanho do ponteiro (8 bytes em 64-bit), não do nome. O cálculo `fseek(pos * sizeof(Cliente))` quebraria silenciosamente.

---

### `db.h` / `db.c` — Data Access Layer

**Único módulo que usa funções de arquivo.** Nenhum outro `.c` do projeto chama `fopen`, `fread`, `fwrite`, `fseek` ou `rewind` diretamente.

Retorna `DbStatus` (enum) em vez de expor detalhes internos:

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

Funções auxiliares (`db_abrir`, `offset_de`) são declaradas `static` — invisíveis fora de `db.c`, equivalente a métodos `private`.

A função `db_listar` usa o padrão **callback/iterator**:
```c
typedef void (*db_iter_fn)(const Cliente *c, int pos, void *userdata);
int db_listar(db_iter_fn callback, void *userdata);
```
Isso permite que `menu.c` defina *como* exibir cada registro sem que `db.c` precise conhecer a lógica de formatação.

---

### `validacao.h` / `validacao.c` — Camada de Input

Não depende de nenhum outro módulo do projeto. Centraliza toda leitura de `stdin`, garantindo que o programa nunca trave em entrada inválida:

- `val_ler_int()` — repete até receber um inteiro válido
- `val_ler_double()` — repete até receber um número válido
- `val_ler_string()` — usa `fgets`, remove `\n`, rejeita strings vazias
- `val_numero_conta()` / `val_saldo()` — regras de negócio de validação

---

### `menu.h` / `menu.c` — Apresentação e Controller

Orquestra as operações CRUD chamando `db_*` e `val_*`. Toda formatação de output (`printf` de resultados) vive aqui. Interpreta `DbStatus` e exibe mensagens amigáveis ao usuário via `imprimir_status()`.

A função pública é apenas uma: `menu_executar()`. O `main.c` não precisa saber nada sobre como o menu funciona internamente.

---

### `main.c` — Entry Point

Com poucas linhas e responsabilidade única: exibir o banner inicial e chamar `menu_executar()`. Nenhuma lógica de negócio.

---

## Modelo de Dados

O arquivo `contas.dat` é um arquivo binário de **registros de tamanho fixo**. Cada registro ocupa exatamente `sizeof(Cliente)` bytes (tipicamente 72 bytes em sistemas de 64-bit com alinhamento padrão).

```
Byte 0                                          Byte 71
┌──────────┬────────────────────────┬───────────┬──────────┐
│  numero  │         nome           │   saldo   │  ativo   │
│  4 bytes │       50 bytes         │  8 bytes  │  4 bytes │
└──────────┴────────────────────────┴───────────┴──────────┘
```

**Exclusão lógica:** contas encerradas têm `ativo = 0` mas o slot permanece no arquivo. Na próxima inserção, o sistema varre o arquivo em busca do primeiro slot inativo para reutilizá-lo antes de fazer append.

---

## Operações de Arquivo Binário

| Função | Onde é usada | Finalidade |
|--------|-------------|------------|
| `fopen("r+b")` | `db_abrir()` | Abre para leitura e escrita sem truncar |
| `fopen("w+b")` | `db_abrir()` (fallback) | Cria o arquivo na primeira execução |
| `fseek(pos * sizeof(Cliente), SEEK_SET)` | `db_buscar`, `db_inserir`, `db_atualizar` | Acesso direto ao registro pelo índice |
| `fread(&c, sizeof(Cliente), 1, fp)` | Todas as operações de leitura | Lê o struct completo de uma vez |
| `fwrite(&c, sizeof(Cliente), 1, fp)` | `db_inserir`, `db_atualizar` | Grava ou sobrescreve um registro |
| `rewind(fp)` | `db_listar()` | Retorna ao byte 0 antes da varredura sequencial |
| `ftell(fp)` | `db_total_registros()` | Calcula quantos registros existem pelo tamanho do arquivo |

---

## Diagrama de Dependências

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

Regra fundamental: **nenhuma dependência circular**. `db.c` não conhece `menu.c`. `validacao.c` não conhece ninguém. `cliente.h` não inclui nenhum header do projeto.

---

## Como Compilar e Executar

### Pré-requisitos

- GCC com suporte a C11 (`gcc --version`)
- Make (`make --version`)
- Linux, macOS ou Windows com MSYS2/MinGW

### Comandos

```bash
git clone https://github.com/Luisr-nunes/Sistema-de-Contas-Bancarias.git
cd Sistema-de-Contas-Bancarias/banco

# Compilar
make

# Compilar e executar imediatamente
make run

# Remover artefatos de build e arquivo de dados
make clean

# Rebuild completo
make rebuild
```

O executável é gerado em `build/banco` (ou `build/banco.exe` no Windows).

### `.gitignore` recomendado

Conteúdo do `.gitignore` do projeto:

```
build/
contas.dat
*.exe
*.o
```

---

## Decisões Técnicas

**Por que `static` nas funções internas de `db.c`?**
`static` em escopo de arquivo impede que o símbolo seja visível fora da unidade de compilação. É o equivalente C de `private`. Sem isso, `db_abrir()` seria acessível a qualquer `.c` do projeto, quebrando o encapsulamento da camada de dados.

**Por que reuso de slots inativos em vez de compactar o arquivo?**
Compactar exigiria reescrever todos os registros posteriores ao deletado — uma operação O(n) potencialmente lenta e com risco de corrupção se interrompida. O reuso de slots mantém a operação em O(n) na busca, mas O(1) na escrita, e é o padrão usado em bancos de dados reais (onde também se chama de "tombstone deletion").

**Por que `double` para saldo em vez de `int` (centavos)?**
Para fins didáticos, `double` é mais direto. Em sistemas financeiros reais, `double` tem problemas de precisão de ponto flutuante (`0.1 + 0.2 ≠ 0.3` em IEEE 754). A solução correta seria armazenar o valor em centavos como `long int` e formatar a exibição dividindo por 100.

**Por que `db_iter_fn` (callback) em `db_listar`?**
O padrão Iterator com callback desacopla *o que buscar* (responsabilidade de `db.c`) de *o que fazer com cada resultado* (responsabilidade de `menu.c`). Se amanhã precisar exportar para CSV, basta passar um callback diferente sem modificar `db.c`.

---

> Desenvolvido como projeto acadêmico para a disciplina de Programação Imperativa e Funcional — CESAR School, Recife/PE.
## Autor

Luís Nunes — [GitHub](https://github.com/Luisr-nunes) · [LinkedIn](https://www.linkedin.com/in/luis-nunes-dev/)
