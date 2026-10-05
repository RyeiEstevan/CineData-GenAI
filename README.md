# CineData GEN-AI: Agente Text-to-SQL

Agente que responde, em português, perguntas sobre o catálogo de filmes da CineData. Ele transforma a pergunta em SQL, executa em modo **somente leitura** na camada Gold (`cinerocket.db`) e explica o resultado, para quem não sabe SQL.

**Stack:** Python 3.12+ · SDK `openai` apontando para o OpenRouter (modelos gratuitos) · SQLite · pandas · matplotlib · Jupyter Notebook

> O notebook é entregue **sem saídas salvas**: ele depende do banco e da sua chave do OpenRouter. As partes offline (seções 1 a 4 e 8) foram testadas com um banco sintético de mesmo schema.

## Como funciona

```
Pergunta → guardrail prévio → cache → modelo (OpenRouter) ⇄ ferramenta executar_sql → banco (ro) → resposta + SQLs + gráfico
```

1. **Guardrail prévio:** pedidos óbvios de injection ("ignore suas instruções", `DROP TABLE`, `' OR 1=1`) são barrados localmente, sem gastar requisição
2. **Cache:** pergunta já respondida (com o mesmo prompt) volta do disco
3. **Loop do agente:** o modelo recebe o schema, as regras de negócio e a ferramenta `executar_sql`. Se o SQL der erro, o erro volta para o modelo corrigir
4. **Fallback:** se um modelo falhar (lotado, fora do ar), o próximo da lista é tentado
5. O notebook mostra a resposta, o modelo usado, as requisições gastas, os SQLs executados (✅/❌) e, se pedido, um gráfico

Uma pergunta típica gasta 2 requisições (uma para gerar o SQL, outra para a resposta).

## Como rodar

### Pré-requisitos

- Python 3.12 ou superior
- Chave gratuita do [OpenRouter](https://openrouter.ai/keys) (sem cartão)
- O arquivo `cinerocket.db`, da pasta compartilhada da atividade

### Passo a passo

```bash
# 1. Ambiente e dependências
python -m venv .venv
source .venv/bin/activate          # Windows (PowerShell): .venv\Scripts\Activate.ps1
pip install -r requirements.txt

# 2. Chave do OpenRouter
cp .env.example .env               # Windows (PowerShell): Copy-Item .env.example .env
# abra o .env e troque sk-or-v1-sua-chave-aqui pela sua chave

# 3. Banco: coloque o arquivo em BD/cinerocket.db
mkdir -p BD

# 4. Abrir o notebook
jupyter lab cinedata_agente_sql.ipynb
```

No VSCode, abra o notebook e escolha o kernel do `.venv`. Depois use *Run All* (ou rode célula a célula).

> ⚠️ **Cota:** a conta gratuita permite **50 requisições por dia** em modelos gratuitos. As células que gastam cota estão marcadas com a tag `gasta-cota` e usam o cache. As 14 perguntas do desafio gastam cerca de 30 requisições na primeira vez. As seções 1 a 4 e 8 não gastam nada. `verificar_cota()` mostra quanto resta (a cota zera às 21h, horário de Brasília).

### Fazendo suas perguntas

```python
perguntar("Quais os 10 filmes de animação com maior receita?", grafico=True)
perguntar("E só os lançados depois de 2020?", continuar=True)   # continua a conversa
avaliar()                                                       # avaliação automática (seção 10)
```

### Problemas comuns

- **`Todos os modelos falharam (429)`:** o OpenRouter usa 429 para "modelo lotado" e para "cota esgotada". Rode `verificar_cota()`; se ainda houver cota, tente de novo em alguns minutos
- **Nome de modelo não encontrado (404):** os modelos gratuitos mudam com o tempo. Confira a lista em [openrouter.ai/models](https://openrouter.ai/models?supported_parameters=tools&max_price=0) e troque `OPENROUTER_MODELS` no `.env`
- **Banco não encontrado:** coloque o arquivo em `data/cinerocket.db` ou defina `CINEDATA_DB` no `.env`

## Estrutura

```
├── notebooks/cinedata_agente_sql.ipynb   # agente, exploração, gabarito, testes e avaliação
├── BD/cinerocket.db          # banco SQLite (não versionado)
├── notebooks/cache/respostas.json        # cache de respostas (criado ao rodar, não versionado)
├── .env.example                # modelo do .env (a chave real fica no .env, ignorado pelo git)
├── requirements.txt
└── README.md
```

| Seção do notebook | Conteúdo | Gasta cota? |
|---|---|---|
| 1 e 2 | Setup, conexão segura e camada semântica (views) | Não |
| 3 | Exploração e qualidade dos dados, com gráficos | Não |
| 4 | Gabarito: as 14 perguntas do desafio em SQL escrito à mão | Não |
| 5 a 7 | Configuração, prompt de sistema e agente | Não |
| 8 | Testes offline com cliente falso | Não |
| 9 | Perguntas do desafio, extras, memória e guardrails | **Sim** |
| 10 | Avaliação automática contra o gabarito | **Sim** (usa cache) |

## Principais decisões

| Decisão | Por quê |
|---|---|
| **Loop de agente próprio** (SDK `openai` + OpenRouter), sem framework | O ciclo é pequeno (chamar modelo, rodar ferramenta, devolver resultado) e escrevê-lo deixa claro cada passo, além de dar controle total sobre fallback, contagem de requisições e erros |
| **`max_retries=0` no cliente** | O SDK repete requisições que falham sozinho, e cada repetição gastaria cota sem aparecer. O fallback entre modelos é feito pelo próprio agente |
| **Uma única ferramenta** (`executar_sql`) com o schema no prompt | Poucas tabelas, então o schema cabe no prompt. Evita chamadas extras de "listar tabelas", o que importa com 50 requisições por dia |
| **Schema gerado do banco**, com chaves de junção deduzidas das colunas `sk_*` | Sem erro de digitação. Colunas inúteis (URLs, `idioma_original` vazio) ficam de fora para economizar tokens |
| **Camada semântica em views temporárias** | Regras críticas (lucro só com receita e orçamento, corte de orçamento abaixo de US$ 10 mil na margem, votos mínimos nas notas) ficam dentro do SQL das views, em vez de depender de o modelo lembrar. As views existem só na conexão e não alteram o banco |
| **Guardrails em camadas** | Filtro prévio local (grátis) + prompt + só `SELECT`/`WITH` + **autorizador do SQLite** (só permite leitura, mesmo que o texto passe) + banco em modo `ro` + limite de 50 linhas + timeout de 120 s. As camadas técnicas não dependem de o modelo obedecer |
| **Cache em disco**, com versão ligada ao prompt | Rodar o notebook de novo não gasta cota. Mudou o prompt, o cache antigo deixa de valer |
| **Memória enxuta** (`continuar=True`) | A continuação envia só perguntas e respostas anteriores em texto, sem os resultados das consultas, e lembra no máximo 4 turnos |
| **Teto de 6 requisições por pergunta** | Conta também as tentativas que falharam e protege a cota se um modelo entrar em loop |
| **Resultado em CSV, máx. 50 linhas** | CSV é o formato mais barato em tokens, e o corte evita estourar o contexto |
| **Gabarito em SQL + avaliação automática** | O gabarito usa as tabelas diretamente (não as views), então é um teste independente. A avaliação compara o ranking do agente com o de referência e dá um status por pergunta |
| **Testes com cliente falso** | Autocorreção, fallback, cache, guardrail, teto de requisições e memória são testados sem chamar o OpenRouter |

## Regras de negócio

Descobertas na exploração dos dados (seção 3) e aplicadas no prompt e nas views:

- "Receita", "faturamento" e "bilheteria" são a mesma coisa. Valores em **R$** por padrão. "Nota" sem fonte significa **IMDb**
- **O lucro do banco engana:** vale `-orçamento` sem receita e a própria receita sem orçamento. Por isso o lucro só é calculado com receita **e** orçamento informados (`v_financas`). Quando o pedido é "lucro com receita informada", a view `v_receita_informada` ignora automaticamente quem não tem orçamento, e a resposta informa quantos filmes entraram
- **Margem** = lucro ÷ receita e **retorno (ROI)** = lucro ÷ orçamento, sempre com orçamento de pelo menos US$ 10 mil (`v_financas_margem`)
- **Margem de grupo** (gênero, produtora) = soma do lucro ÷ soma da receita, nunca a média das margens por filme
- Divisões com `* 1.0` no numerador (o SQLite faz divisão inteira)
- **Notas:** TMDB = 0 significa "sem votos". Rankings de divergência exigem votos mínimos (100 no TMDB e no IMDb, 4 avaliações de usuários). Médias de grupos não exigem
- **Pessoas:** o papel está em `dim_people.tipo_pessoa`; contar por `sk_person_id` por causa de homônimos
- **Gêneros em inglês** ("terror" vira `Horror`). **"Últimos N anos"** = do ano atual menos N + 1 até o ano atual

## Qualidade dos dados

A exploração mostra pontos que o agente herda, porque ele responde sobre os dados como estão: receita rara (poucos filmes têm receita e orçamento), orçamentos de cadastro absurdos, títulos repetidos com ids diferentes, popularidade igual ao ano de lançamento em alguns filmes e `idioma_original` vazio.

## Limitações conhecidas

- Modelos gratuitos não são 100% previsíveis: a mesma pergunta pode vir com outra explicação ou ignorar uma regra. Por isso os SQLs executados aparecem sempre junto da resposta
- Modelos gratuitos podem ser lentos ou ficar lotados. A cota é de 50 requisições por dia, e requisições que falham também contam
- A avaliação automática compara só a primeira coluna do resultado (os itens do ranking), não os números. Não substitui ler a resposta
- As views foram desenhadas em cima das perguntas do desafio. Perguntas novas podem cair no SQL direto e dependem mais do prompt

## Próximos passos

- Comparar também as colunas numéricas na avaliação e medir a taxa de acerto por modelo
- Levar mais regras do prompt para views
- Interface de chat (por exemplo, com Streamlit)
- Agente híbrido: busca semântica nas sinopses combinada com SQL
