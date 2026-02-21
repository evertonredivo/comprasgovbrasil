# PNCP API — Guia de Referência para Inteligência de Mercado

## Visão Geral

| Item | Detalhe |
|------|---------|
| **Base URL (Consulta)** | `https://pncp.gov.br/api/consulta/v1` |
| **Base URL (Integração)** | `https://pncp.gov.br/api/pncp/v1` |
| **Swagger UI** | https://pncp.gov.br/api/consulta/swagger-ui/index.html |
| **Autenticação** | Não necessária para consultas (GET) |
| **Formato** | REST / JSON |
| **Rate Limit** | Não documentado oficialmente — usar pausas de 300-500ms entre requisições |

---

## Endpoints Estratégicos

### 1. Contratações por Data de Publicação

Busca licitações publicadas em um período. **Ponto de entrada principal.**

```
GET /v1/contratacoes/publicacao
```

| Parâmetro | Obrigatório | Exemplo | Descrição |
|-----------|:-----------:|---------|-----------|
| `dataInicial` | ✅ | `20250101` | Formato YYYYMMDD |
| `dataFinal` | ✅ | `20250221` | Formato YYYYMMDD |
| `codigoModalidadeContratacao` | | `6` | 6=Pregão Eletrônico, 8=Dispensa |
| `uf` | | `SC` | Filtra por estado |
| `codigoMunicipioIbge` | | `4218004` | Código IBGE do município |
| `cnpj` | | `83102459000155` | CNPJ do órgão |
| `pagina` | | `1` | Página de resultados |
| `tamanhoPagina` | | `50` | Registros por página |

**Exemplo prático:**
```
https://pncp.gov.br/api/consulta/v1/contratacoes/publicacao?dataInicial=20250101&dataFinal=20250221&uf=SC&codigoModalidadeContratacao=6&pagina=1
```

**O que retorna:** Lista de contratações com `cnpj`, `anoCompra`, `sequencialCompra`, `objetoCompra`, `valorTotalEstimado`, etc.

---

### 2. Editais com Propostas Abertas (OPORTUNIDADES ATIVAS!)

```
GET /v1/contratacoes/proposta
```

| Parâmetro | Obrigatório | Exemplo | Descrição |
|-----------|:-----------:|---------|-----------|
| `dataFinal` | ✅ | `20250320` | Buscar até esta data |
| `uf` | | `SC` | Estado |
| `codigoModalidadeContratacao` | | `6` | Modalidade |

**Use na rotina diária 7h:** Este endpoint mostra editais onde você AINDA PODE participar.

---

### 3. Itens de uma Contratação

Dado o CNPJ/ano/sequencial de uma contratação, retorna TODOS os itens.

```
GET /v1/orgaos/{cnpj}/compras/{ano}/{sequencial}/itens
```

**Exemplo:**
```
https://pncp.gov.br/api/pncp/v1/orgaos/83102459000155/compras/2025/42/itens
```

**O que retorna:** Array com cada item: `descricao`, `quantidade`, `unidadeMedida`, `valorUnitarioEstimado`, `valorTotalEstimado`, `situacaoCompraItemId` (2=Homologado), `tipoBeneficioId` (1=Exclusivo ME/EPP).

---

### 4. Resultado de um Item (PREÇO VENCEDOR! 💰)

**Este é o endpoint mais valioso para pesquisa de preços.**

```
GET /v1/orgaos/{cnpj}/compras/{ano}/{sequencial}/itens/{numeroItem}/resultados
```

**Retorna:**
```json
{
  "valorUnitarioHomologado": 24.50,
  "valorTotalHomologado": 2450.00,
  "quantidadeHomologada": 100,
  "niFornecedor": "12345678000199",
  "nomeRazaoSocialFornecedor": "Empresa XYZ Ltda",
  "porteFornecedorId": 1,
  "dataResultado": "2025-01-15"
}
```

**Porte do fornecedor:** 1=ME, 2=EPP, 3=Demais → Saber quem são seus concorrentes diretos!

---

### 5. Atas de Registro de Preço Vigentes

```
GET /v1/atas?dataInicial=20250221&dataFinal=20260221&pagina=1
```

Atas vigentes mostram preços já registrados que órgãos podem aderir.

---

### 6. Plano de Contratações Anual (PCA) — PREVISÃO DE DEMANDA

```
GET /v1/pca/?anoPca=2026&pagina=1
```

**Ouro puro:** Mostra o que os órgãos planejam comprar. Você descobre a demanda ANTES do edital sair.

---

## Tabelas de Domínio (Códigos)

### Modalidade de Contratação
| Código | Modalidade |
|:------:|-----------|
| 4 | Concorrência Eletrônica |
| 6 | **Pregão Eletrônico** ← seu foco principal |
| 7 | Pregão Presencial |
| 8 | **Dispensa de Licitação** ← quick wins |
| 9 | Inexigibilidade |

### Situação do Item
| Código | Situação |
|:------:|----------|
| 1 | Em andamento |
| 2 | **Homologado** ← tem preço vencedor |
| 3 | Anulado/Revogado |
| 4 | Deserto |
| 5 | Fracassado |

### Tipo de Benefício
| Código | Tipo |
|:------:|------|
| 1 | **Exclusivo ME/EPP** ← itens onde só MEI/ME/EPP participam |
| 2 | Subcontratação ME/EPP |
| 3 | **Cota reservada ME/EPP** ← 25% reservado para pequenos |
| 4 | Sem benefício |

### Porte do Fornecedor
| Código | Porte |
|:------:|-------|
| 1 | **ME (Microempresa)** ← inclui MEI |
| 2 | **EPP (Empresa Pequeno Porte)** |
| 3 | Demais |

---

## Fluxo de Pesquisa de Preços (Passo a Passo)

```
1. Buscar Contratações          → GET /contratacoes/publicacao?uf=SC&modalidade=6
       ↓
   Retorna: cnpj, ano, sequencial de cada pregão
       ↓
2. Buscar Itens                 → GET /orgaos/{cnpj}/compras/{ano}/{seq}/itens
       ↓
   Filtrar: itens com "papel a4" na descrição
       ↓
3. Buscar Resultado             → GET /orgaos/{cnpj}/compras/{ano}/{seq}/itens/{num}/resultados
       ↓
   Retorna: valorUnitarioHomologado (PREÇO VENCEDOR!)
       ↓
4. Calcular Estatísticas        → mediana, menor, maior, desvio padrão
       ↓
5. Preencher Planilha           → Colunas F, G, H da sua Matriz de Inteligência
```

---

## Códigos IBGE — Municípios Alvo (Raio ~100km Tubarão)

| Município | Código IBGE |
|-----------|:-----------:|
| Tubarão | 4218004 |
| Laguna | 4209409 |
| Criciúma | 4204608 |
| Içara | 4207007 |
| Capivari de Baixo | 4203600 |
| Braço do Norte | 4202909 |
| Imbituba | 4207304 |
| Gravatal | 4206306 |
| Jaguaruna | 4208906 |
| Orleans | 4211702 |
| Urussanga | 4219002 |
| Florianópolis | 4205407 |
| Joinville | 4209102 |

---

## Códigos CATMAT — Seus Produtos Estratégicos

| Produto | CATMAT |
|---------|:------:|
| Papel A4 75g (resma 500 fls) | 467347 |
| Toner HP CF283A (83A) | 150710 |
| Toner HP CF226A (26A) | 448858 |
| Toner HP CF280A (80A) | 336186 |
| Mouse USB | 326267 |
| Teclado USB ABNT2 | 326291 |
| Pen Drive 64GB | 483051 |
| Caneta esferográfica | 203518 |
| Clips 2/0 | 271776 |
| Grampo 26/6 | 261513 |

---

## Como Usar o Script

### Pré-requisitos
- Node.js 18 ou superior (usa `fetch` nativo)
- Conexão com internet

### Instalação
```bash
# Nenhuma dependência externa necessária!
# Basta ter o Node.js 18+

# Verificar versão
node --version  # deve ser v18.x ou superior
```

### Comandos Principais

```bash
# 1. PESQUISA DE PREÇOS (preenche a planilha!)
node pncp-inteligencia.js --pesquisa "papel a4"
node pncp-inteligencia.js --pesquisa "toner"
node pncp-inteligencia.js --pesquisa "mouse"

# 2. OPORTUNIDADES ABERTAS (rotina diária 7h)
node pncp-inteligencia.js --propostas

# 3. VER CONTRATAÇÕES RECENTES EM SC
node pncp-inteligencia.js --contratacoes

# 4. ANALISAR UM PREGÃO ESPECÍFICO
# (use CNPJ/ano/sequencial que aparece nas listagens)
node pncp-inteligencia.js --analisar 83102459000155 2025 42

# 5. PLANOS DE CONTRATAÇÃO 2026 (demanda futura)
node pncp-inteligencia.js --pca 2026
```

### Saída Típica da Pesquisa de Preços

```
═══════════════════════════════════════════════════════
  📊 RESULTADO DA PESQUISA: "papel a4"
═══════════════════════════════════════════════════════
  Amostras:        23 preços homologados
  ─────────────────────────────────────────
  Menor preço:     R$ 22,00
  Mediana:         R$ 27,50  ← USE COMO REFERÊNCIA
  Média:           R$ 28,30
  Maior preço:     R$ 35,00
  ─────────────────────────────────────────
  Desvio padrão:   R$ 3,20
  Coef. variação:  11.3% ✅ Mercado estável
═══════════════════════════════════════════════════════

  📋 VALORES PARA SUA PLANILHA:
  ┌─────────────────────────────────────────┐
  │ Coluna F (Mediana Gov):       R$ 27,50  │
  │ Coluna G (Menor Preço):      R$ 22,00  │
  │ Coluna H (Maior Preço):      R$ 35,00  │
  └─────────────────────────────────────────┘
```

---

## Automação — Rotina Diária

Crie um script bash para executar automaticamente:

```bash
#!/bin/bash
# rotina-diaria.sh — Executar todo dia às 7h

echo "=== ROTINA DIÁRIA PNCP — $(date) ==="

# 1. Verificar editais abertos
node pncp-inteligencia.js --propostas > relatorio_$(date +%Y%m%d).txt

# 2. Atualizar pesquisa de preços dos 5 itens principais
for item in "papel a4" "toner hp" "mouse usb" "caneta esferografica" "pen drive"; do
  echo "Pesquisando: $item"
  node pncp-inteligencia.js --pesquisa "$item" >> precos_$(date +%Y%m%d).txt
  sleep 5  # pausa entre pesquisas
done

echo "=== CONCLUÍDO ==="
```

Para agendar via cron:
```bash
crontab -e
# Adicionar:
0 7 * * * /caminho/para/rotina-diaria.sh
```

---

## Uso como Biblioteca (para integrar em outros projetos)

```javascript
const pncp = require('./pncp-inteligencia.js');

// Pesquisar preços
const resultado = await pncp.pesquisaPrecos('papel a4');
console.log(resultado.estatisticas.mediana); // R$ 27.50

// Buscar editais abertos
const editais = await pncp.buscarPropostasAbertas({ uf: 'SC' });

// Analisar contratação específica
const itens = await pncp.buscarItensContratacao('83102459000155', '2025', '42');
```
