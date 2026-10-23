# ☕ Vending Machine — Autômatos Finitos e Mini-scanner

**Disciplina:** Compiladores — Prática 3 (Autômatos Finitos)

**Integrantes:** [preencher com os nomes de todos os integrantes do grupo]

**Tema:** validadores e linguagem de comandos para uma máquina de café / vending machine.

## 1. Problema e objetivo

Uma máquina de café ou vending machine precisa validar códigos de produtos, aceitar apenas denominações de moedas previstas e controlar as etapas de uma compra. Entradas inválidas — como códigos fora do catálogo, moedas não aceitas ou a seleção de um produto antes de haver crédito suficiente — precisam ser rejeitadas de forma previsível. Este projeto aplica conceitos de Linguagens Formais e Autômatos para reconhecer códigos de produtos e valores monetários, modelar o acúmulo de crédito como uma máquina de estados finitos e analisar comandos textuais como `INSERIR 1.00`, `ESCOLHER B3` e `TROCO`.

O notebook reutiliza a classe `AFD` e o exemplo de scanner manual da aula, acrescentando novas células ao final sem apagar as células originais.

## 2. AFDs do domínio

### 2.1 Validador de código de produto

O código é formado por uma letra maiúscula de `A` a `F` e um algarismo de `1` a `9`. Existem cinco estados explícitos: estado inicial, dois estados intermediários que separam as classes de letras e dois estados finais.

```text
                         -- 1-9 --> ((qf_ABC))
                        /
→ (q0) -- A-C --> (q_ABC)
  |
  +---- D-F --> (q_DEF) -- 1-9 --> ((qf_DEF))
```

**Expressão regular equivalente:**

```regex
[A-F][1-9]
```

Exemplos aceitos: `A1`, `B3`, `C9`, `D5`, `F9`.

Exemplos rejeitados: palavra vazia, `A0`, `G1`, `A10`, `a1`.

### 2.2 Validador de moedas

As denominações permitidas são `0.25`, `0.50` e `1.00`. Para a bateria AFD × RegEx ter mais de cinco entradas aceitas sem adicionar novas denominações, o validador aceita zeros à esquerda como representações equivalentes: por exemplo, `00.25` tem o mesmo valor numérico que `0.25`, e `01.00` equivale a `1.00`.

```text
→ q0 --0--> qZ (loop em 0)
             |-- . --> qZdot --2--> q25 --5--> ((qf25))
             |                    `--5--> q50 --0--> ((qf50))
             `--1--> q1 -- . --> q1dot --0--> q100 --0--> ((qf100))
```

**Expressão regular equivalente:**

```regex
0*(?:0\.(?:25|50)|1\.00)
```

Exemplos aceitos: `0.25`, `00.25`, `000.25`, `0.50`, `00.50`, `1.00`, `01.00`.

Exemplos rejeitados: palavra vazia, `0.00`, `0.20`, `0.75`, `1.50`, `2.00`, `0,25`.

> Nos dois validadores, a bateria usa `re.fullmatch`, exigindo que a palavra inteira seja reconhecida. O notebook imprime a tabela `tabela()` de cada AFD e rastreia palavras aceitas e rejeitadas. A rejeição pode acontecer por falta de transição (estado morto no meio da palavra) ou por terminar fora do conjunto de estados finais.

## 3. Máquina de estados do negócio

A vending machine acompanha o crédito em passos de R$ 0,25, de `C0` (R$ 0,00) até `C150` (R$ 1,50). O produto de demonstração `B3` custa R$ 1,25. Com crédito suficiente, o evento `ESCOLHER B3` leva a `LIBERANDO`; o evento `TROCO` termina a venda e calcula o retorno.

```text
C0 --0.25--> C025 --0.25--> C050 --0.25--> C075 --0.25--> C100 --0.25--> C125 --0.25--> C150
 |             |              |              |              |              |              |
 +--0.50-----> C050           +--0.50-------> C100          +--0.50-------> C150          +--ESCOLHER B3--> LIBERANDO
 +--1.00-----> C100           +--1.00-------> C150                         C125 --ESCOLHER B3--> LIBERANDO
                                                                            LIBERANDO --TROCO--> TROCO
```

A máquina tem **9 estados e 17 transições** no modelo implementado. O limite de R$ 1,50 mantém este exemplo finito; ao tentar inserir outra moeda quando o limite já foi atingido, o autômato rejeita o evento e mostra o evento permitido (`ESCOLHER B3`).

### Fluxos válidos

**Fluxo 1 — crédito acima do preço, com troco:**

```text
INSERIR 1.00
INSERIR 0.50
ESCOLHER B3
TROCO
```

Saída resumida: crédito `R$ 1.50`, produto `B3` liberado, troco `R$ 0.25`, venda concluída.

**Fluxo 2 — crédito exato:**

```text
INSERIR 0.50
INSERIR 0.50
INSERIR 0.25
ESCOLHER B3
TROCO
```

Saída resumida: crédito `R$ 1.25`, produto `B3` liberado, troco `R$ 0.00`, venda concluída.

### Fluxos inválidos

- `ESCOLHER B3` no estado `C0`: recusado porque ainda não há crédito. O programa exibe como permitidos `INSERIR 0.25`, `INSERIR 0.50` e `INSERIR 1.00`.
- `INSERIR 1.00` → `INSERIR 0.50` → `INSERIR 0.25`: a terceira moeda é recusada no estado `C150`, pois ultrapassaria o limite definido para a demonstração. O evento permitido passa a ser `ESCOLHER B3`.

## 4. Mini-scanner e tabela de tokens

O scanner foi implementado manualmente a partir do exemplo `escanear()` da aula. Ele lê a entrada da esquerda para a direita, reconhece tokens, ignora espaços e comentários iniciados por `#`, registra erros com linha e coluna e continua a varredura depois de encontrar um caractere inválido.

| Token | Descrição | Padrão / exemplo |
|---|---|---|
| `INSERIR` | Palavra reservada para inserir crédito | `INSERIR` |
| `ESCOLHER` | Palavra reservada para selecionar um produto | `ESCOLHER` |
| `TROCO` | Palavra reservada para concluir e calcular o troco | `TROCO` |
| `COD_PRODUTO` | Lexema com formato de letra seguida de dígito; o AFD verifica se está no intervalo | `B3`, `G3` |
| `MOEDA` | Uma das denominações aceitas, com zeros à esquerda opcionais | `0.25`, `0.50`, `1.00` |
| `NUM_INT` | Número inteiro | `1`, `25` |
| `NUM_REAL` | Número decimal sintaticamente formado, mas fora das denominações aceitas; também gera dica | `0.75`, `2.00` |
| `OP_COMP` | Operadores relacionais | `==`, `<=`, `>=`, `!=`, `<`, `>` |
| `ATRIB` | Operador de atribuição simples | `=` |
| `PAREN_ESQ` | Parêntese de abertura | `(` |
| `PAREN_DIR` | Parêntese de fechamento | `)` |
| `PONTO_VIRGULA` | Separador | `;` |
| `MAIS` | Operador de adição | `+` |
| `MENOS` | Operador de subtração | `-` |
| `ID` | Identificador não reservado | `produto` |

### Maximal Munch

O scanner testa os operadores de dois caracteres primeiro. Por isso, `==` é um único `OP_COMP`, enquanto `=` é `ATRIB`; da mesma forma, `<=` e `<` são reconhecidos separadamente. A entrada `1.00 1` também mostra a distinção entre `MOEDA` e `NUM_INT`.

Exemplo de comandos:

```text
# compra de café
INSERIR 1.00
INSERIR 0.50
ESCOLHER B3
TROCO
```

Exemplo de tokens produzidos (linhas/colunas dependem das quebras de linha da entrada):

```text
INSERIR      -> INSERIR
MOEDA        -> 1.00
INSERIR      -> INSERIR
MOEDA        -> 0.50
ESCOLHER     -> ESCOLHER
COD_PRODUTO  -> B3
TROCO        -> TROCO
```

### Tratamento de erros

Dicas específicas incluídas no scanner:

- Vírgula decimal: `use ponto decimal para moedas, por exemplo 0.50.`
- Símbolo `R$`: `não inclua o símbolo R$; informe apenas 0.25, 0.50 ou 1.00.`
- Comando desconhecido como `@`: `comando desconhecido; use INSERIR, ESCOLHER ou TROCO.`
- Valor decimal não aceito como `0.75`: informa o valor inválido e apresenta as denominações permitidas.

O scanner avança depois do erro, portanto também consegue reconhecer `ESCOLHER B3` e `TROCO` se aparecerem nas linhas seguintes.

## 5. Interface interativa

O notebook inclui um painel `ipywidgets` com três modos:

1. validar código de produto;
2. validar moeda;
3. executar o mini-scanner e visualizar os tokens e erros.

## 6. Como executar

### Google Colab

1. Faça upload de `Aula08_Pratica3_VendingMachine_Completo.ipynb` ou abra o notebook no Drive.
2. Execute as células em ordem, incluindo as novas células R1–R6 ao final.
3. Use o painel interativo na última seção para testar entradas.

### VS Code

1. Instale as extensões **Python** e **Jupyter**.
2. Instale as dependências do painel:

   ```bash
   python -m pip install ipywidgets ipykernel jupyter
   ```

3. Abra o arquivo `.ipynb`, selecione o interpretador Python adequado e execute as células em ordem.

O núcleo dos autômatos e do scanner usa `re` e `decimal`, que fazem parte da biblioteca padrão do Python; `ipywidgets` e `IPython` são usados pela interface do notebook.

## 7. Conceitos demonstrados

- Autômato Finito Determinístico (AFD) e tabela de transição;
- classes de símbolos e estados finais;
- comparação de AFD e expressão regular com zero divergências;
- rastreamento passo a passo de palavra aceita e rejeitada;
- máquina de estados de negócio e eventos permitidos;
- Maximal Munch;
- reconhecimento de tokens;
- comentários e espaços ignorados;
- erros com linha, coluna e dicas específicas;
- painel interativo para testes.

## Integrantes

- [Nome completo do integrante 1]
- [Nome completo do integrante 2]
- [Nome completo do integrante 3]
- [Adicionar ou remover linhas conforme o tamanho do grupo]
