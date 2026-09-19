# Spout Finance — Notas Cruas (Beta Intelligence Challenge)

Status: Dia 0 — varredura de docs completa, aguardando primeira sessão no app.
Beta code obtido: [09/09/2026]
Wallet de teste: DHG4p1tKiXuQS2oYUMAnxR1P4YDgzGdkQfzJZfYoRnNV

---

## Perfil do reviewer (pra Executive Summary)

- Solana dev full-stack (Rust/Anchor), bounty hunter Superteam Brasil
- Já testei lending/borrow DeFi (2º lugar Zodial — Superteam Germany)
- Approach: sem seguir tutorial oficial na primeira sessão, documentar hesitação real

---

## FPs pré-carregados da varredura de documentação (validar contra o app)

### FP-DOC-1 — Yield alvo diverge entre páginas dos docs

**What happened:** `/introduction` e `/how-lending-works` mostram Senior ~9% / Junior ~32% APY. `/what` mostra Senior ~7% / Junior ~25%+. A matemática detalhada em `/lending-tranches` confirma que 9%/32.8% são os números reais (7% é só a "priority yield" antes do excess split).

**Why it caused friction:** Um usuário que lê só `/what` (página de overview) forma expectativa de retorno abaixo do real — ou, pior, se ler só a landing page, acima do que a página educacional mais detalhada sugere. Cost/yield disclosure inconsistente é exatamente o tipo de achado que rendeu 1º lugar ao Minkhanov na Zodial.

**Severity:** High (candidato a Critical se a UI do app também usar o número de 7%)

**Suggested improvement:** Padronizar todas as menções de APY pra usar o número blended (9%/32%), com nota clara de que a "priority yield" (7%) é só o piso garantido do Senior antes do split de excess.

**TODO:** verificar qual número aparece na tela de deposit do app.

---

### FP-DOC-2 — Liquidation fee: 5% flat (marketing) vs 8.8% no exemplo prático

**What happened:** `/liquidation` e `/fee-structure` afirmam fee flat de 5%, comparando favoravelmente aos "10-15% padrão DeFi". Mas o cenário do Dave em `/scenarios` usa "NVDA's liquidation fee (8.8%, its per-asset buffer)" — linguagem que sugere fee variável por ativo, não flat.

**Why it caused friction:** Se a fee real varia por volatilidade do ativo (o que faz sentido do ponto de vista de risco), a claim de marketing "5% flat, melhor que o mercado" pode estar escondendo que ativos voláteis (NVDA, MSTR) pagam quase o dobro do anunciado.

**Severity:** Critical (afeta diretamente a decisão de quanto colateral trazer e qual ativo escolher)

**Suggested improvement:** Página de fees deveria mostrar a fee por ativo (como já faz pra LTV/cycle na tabela de `/supported-collateral`), não só um número flat genérico.

**TODO:** simular/observar liquidação (ou tela de preview de liquidação) em pelo menos 2 ativos de volatilidade diferente (ex: NVDA/MSTR vs PFE/GLD) e comparar a fee mostrada.

---

### FP-DOC-3 — "Keep every share, no losing your upside" vs mecânica real de assignment

**What happened:** Landing page / `/introduction`: "No selling, no taxable event, no losing your upside." Mas `/covered-call-strategy` e `/options-assignment` deixam claro que, em caso de assignment, as ações SÃO vendidas no strike — o upside acima do strike naquele ciclo é sacrificado. O FAQ chama isso de "bounded outcome", mas a claim da landing page ignora esse cenário por completo.

**Why it caused friction:** Um usuário que só lê a landing page (o funil de entrada de qualquer produto) forma uma expectativa de risco zero de upside que não é verdade. Isso é "leverage/risk framing" — exatamente o ângulo que diferenciou o 1º lugar do 2º na Zodial.

**Severity:** High

**Suggested improvement:** Landing page deveria trocar "no losing your upside" por algo como "no losing your upside below the strike" ou linkar diretamente pro FAQ de assignment.

**TODO:** verificar se o app, no fluxo de lock/borrow, avisa sobre o risco de assignment ANTES do usuário confirmar a transação, ou só depois nos docs.

---

### FP-DOC-4 — "LP reserve" mencionado sem explicação em nenhum lugar

**What happened:** `/distribution` menciona dedução de "protocol fee, insurance fund contribution, and the LP reserve" antes da distribuição pro lender. Mas `/settlement-flow` (página mais detalhada do mesmo fluxo) só lista protocol fee + insurance fund + senior priority + excess split — sem "LP reserve". O termo não aparece no glossário.

**Why it caused friction:** Termo técnico não documentado que afeta diretamente quanto o lender recebe. Zero explicação de tamanho, propósito ou regra de acúmulo.

**Severity:** Medium

**Suggested improvement:** Adicionar "LP reserve" ao glossário e reconciliar as duas páginas pra descreverem o mesmo fluxo de settlement de forma idêntica.

**TODO:** checar tela de distribution no app — o termo aparece ali? Em que valor?

---

### FP-DOC-5 — Circuit breaker sem threshold público

**What happened:** `/circuit-breakers`: "If the fund draws down past a defined threshold, new cycles for affected assets pause." O número nunca é revelado em nenhuma página.

**Why it caused friction:** É um mecanismo de segurança estrutural importante (o que acontece quando o Insurance Fund já drenou bastante) mas o usuário não tem como avaliar quão perto desse limite o protocolo está a qualquer momento.

**Severity:** Medium

**Suggested improvement:** Publicar o threshold numérico (ex: "pausa quando o fundo cai abaixo de X% do target") e mostrar isso na UI de transparência do fundo.

**TODO:** ver se o app mostra saldo/histórico do Insurance Fund (docs dizem que sim: "current fund balance, target level, contribution rate, and drawdown history are visible in the app").

---

## Ângulos de Financial Safety Analysis (a explorar durante o teste)

1. **Weekend/overnight gap risk**: `/oracles` diz que fora do horário de mercado dos EUA o preço atualiza em "frequência reduzida". Ações têm gap de fim de semana / after-hours. Se NVDA abrir 15% abaixo na segunda, o Health Factor só reage quando o mercado abre? Existe proteção?
2. **Tax withholding 30% pra não-US** (`/tax`) — relevante pro público BR, ângulo de conteúdo específico pra Superteam Brasil.
3. **BSOL como colateral** — exposição a Solana via stock/ETF, ângulo natural pra puxar audiência cripto-nativa ("Onchain Native" persona dos docs).
4. **Sensibilidade real da liquidation fee** — testar com pelo menos 2 ativos de volatilidade diferente.

---

## FP-APP-10 — Recon técnico (DevTools): sem vazamento de chaves, mas polling ineficiente de RPC

**What happened:** Inspeção do Network tab (Fetch/XHR) na tela de Trade mostra chamadas diretas do browser pro RPC público `api.devnet.solana.com`, sem proxy de backend. São majoritariamente `getTokenAccountBalance` repetidas pras mesmas duas contas (`8KTa1mJHsy6UfswXP8HxQBVzcH4jgRkwh3hHqLDLLUFf`, `3rAWFGFUitzCXCouzU3fdwdYCaCBfSVCX3VYogyGeMc8`) mais `getAccountInfo` e `getMultipleAccounts`, todas repetindo a cada ~140-160ms.

**Segurança (positivo):** Nenhuma API key ou secret visível nesses headers/payloads — faz sentido, pois RPC devnet público não exige auth. Não há vazamento aqui. Sources tab (busca por 'sk_', 'api_key', 'secret', etc no bundle JS) ainda não verificado.

**Performance (achado técnico, não-crítico):** O padrão é polling ativo repetido em vez de subscription via WebSocket (`onAccountChange`/`accountSubscribe` do `@solana/web3.js`). Funciona bem em devnet com baixa carga, mas não escala — em mainnet, RPC público teria rate-limit rápido nesse volume de requests; mesmo com RPC dedicado (Helius, que é o que o produto provavelmente usará em produção), polling constante é desperdício de créditos de RPC comparado a subscriptions.

**Severity:** Low/Info — não é bug, é sugestão de otimização de arquitetura.

**TODO:** ainda falta checar Sources tab (busca de secrets no bundle) e Application tab (localStorage/cookies) — pendente.

---

## FP-APP-11 — Lighthouse baseline (desktop, /buy) — excelente, e Sources tab sem secrets óbvios

**Lighthouse scores (desktop, beta.spout.finance/buy, 09/09/2026 20:39 GMT-3):**
- Performance: 100 (FCP 0.2s, LCP 0.6s, TBT 20ms, CLS 0, SI 0.4s)
- Accessibility: 100
- Best Practices: 100
- SEO: 91 (único ponto: documento sem meta description)
- Agentic Browsing: 2/2

**Why this matters:** É um resultado muito acima da média DeFi — a maioria dos protocolos que testei antes (Hobba, Zodial) tinha scores de Performance na faixa de 60-80. Vale citar como "what works" forte no report; poucos concorrentes provavelmente vão rodar esse audit, então é um diferencial de rigor técnico.

**Único ponto de melhoria real:** SEO 91 por falta de `<meta name="description">` — quick win fácil de reportar.

## FP-APP-14 — Sinalização de devnet vs. dinheiro real não é clara o suficiente na UI

**What happened:** Depois de um dia inteiro de teste, surgiu a dúvida genuína "coloquei dinheiro real?" — apesar do toast inicial "Wallet verified for devnet" e da seção "DEVNET TEST FUNDS" no painel de wallet, em nenhum momento subsequente da UI (telas de Trade, Borrow, Portfolio) há um indicador persistente e visível de "você está em modo de teste / devnet" enquanto navega e opera.

**Why it caused friction:** Se um desenvolvedor Solana experiente, que literalmente configura infra devnet/mainnet pra viver, teve um momento de "espera, isso é real?", esse é um sinal forte de que o indicador de ambiente é insuficiente pra qualquer usuário menos técnico. O toast de "verified for devnet" aparece uma vez e some — não há badge fixo, cor de tema diferente, ou label persistente (tipo "TESTNET" no header) lembrando o usuário em todas as telas seguintes.

**Severity:** High — isso é sobre confiança e clareza financeira, o núcleo de qualquer produto DeFi. É facilmente resolvível e tem grande impacto de segurança psicológica do usuário.

**Suggested improvement:** Badge persistente no header (ex: "DEVNET" em amarelo/laranja, sempre visível) enquanto o beta estiver rodando em testnet, não só um toast que desaparece. Bandeiras assim são padrão em produtos financeiros de teste (bancos digitais em sandbox, exchanges em modo demo, etc).

---

## FP-APP-12 — Confirmado como achado #1 técnico do report: 500 sistemático em /deposit e /borrow

**What happened:** Na tela de `/borrow`, com a posição de GOOG já ativa, o painel de borrow exibiu a mensagem crua: `CollateralType: unexpected length 213 (expected 165, or 149 pre-migration)`.

**Why this is the strongest technical finding do report:** Essa não é uma mensagem de erro de UI — é literalmente um erro de **desserialização de struct de conta on-chain** (padrão típico de Anchor/Borsh: o parser espera uma conta com um tamanho de bytes específico — 165 bytes no layout atual, ou 149 bytes no layout "pre-migration" — e recebeu 213 bytes, que não bate com nenhum dos dois). Isso indica um de dois cenários sérios:
1. **Migração de schema incompleta**: o programa foi atualizado (migração de layout de conta) mas existem contas antigas/novas com tamanhos inconsistentes sendo lidas pelo mesmo parser, e o "graceful fallback" pro layout pre-migration (149 bytes) não cobre esse caso (213 bytes).
2. **Erro não tratado vazando pro cliente**: mesmo que seja um caso conhecido/esperado no backend, deixar essa string de erro interna (que expõe detalhes de implementação do programa: nomes de campos como `CollateralType`, tamanhos exatos de struct) diretamente na interface é uma prática ruim de error handling — dá info de debug pra qualquer atacante mapeando a estrutura de contas do programa, e assusta/confunde o usuário final que não faz ideia do que isso significa.

**Severity:** Critical — tanto pelo ângulo funcional (o fluxo de borrow pode estar quebrado pra essa conta/colateral específica) quanto pelo ângulo de segurança (vazamento de detalhes de implementação interna, característico do tipo de achado que se busca em auditoria — está diretamente alinhado com "SVS-8" e outros trabalhos anteriores de review de contas Anchor).

**Suggested improvement:** (1) Tratar esse erro no client com uma mensagem humana ("Não foi possível carregar sua posição de colateral — tente novamente ou contate o suporte"), nunca expor a string de erro do parser. (2) No backend/programa, investigar por que essa conta específica (a de colateral GOOG desse usuário, criada nas últimas 24h) está vindo com 213 bytes — se for uma conta recém-criada, não deveria ter esse problema de migração de schema antigo.

**CAUSA RAIZ CONFIRMADA (Console, screenshot 17):** duas chamadas de API falharam com **status 500**:
- `api/vault/deposit?us...YoRnNV&ticker=XOM:1`
- `api/vault/borrow?use...YoRnNV&ticker=XOM:1`

Ambas pro parâmetro `ticker=XOM` — o mesmo ativo errado identificado no FP-APP-13 (o painel abre defaultado pra XOM, ativo que o usuário não possui nenhuma posição). Isso conecta os dois achados numa única causa raiz: **o frontend tenta pré-carregar dados de vault (deposit/borrow) pro ticker default (XOM) antes mesmo do usuário selecionar um ativo; como não existe posição/vault pra esse par usuário+XOM, o endpoint retorna 500; e o client, ao tentar processar essa resposta de erro como se fosse dados de conta, gera o erro de desserialização cru que vaza na tela ("CollateralType: unexpected length...").**

Essa é uma explicação MUITO mais simples e menos alarmante do que a hipótese original de "migração de schema quebrada" — não é um bug de layout de conta on-chain, é um erro de **tratamento de resposta de erro HTTP** no client: um 500 (provavelmente "vault not found" mal categorizado como erro de servidor em vez de 404) sendo processado como se fosse um payload válido de conta.

**Severity revisada:** ainda High (não mais "Critical técnico de blockchain", mas continua sendo um bug real de error handling que vaza detalhes internos e pode confundir/assustar usuários), rebaixado de "possível corrupção de dados on-chain" pra "erro de UX de tratamento de erro HTTP + endpoint retornando código de status errado (500 em vez de 404 pra 'vault não existe')".

**Suggested improvement:** (1) Backend: retornar 404 (não 500) quando o vault não existe pra aquele ticker/usuário — 500 implica erro do servidor, não "recurso não encontrado", que é semanticamente diferente e mais barato de tratar no client. (2) Frontend: nunca inicializar o painel de borrow com uma chamada de API pra um ticker que o usuário não possui — carregar vazio/neutro até o usuário selecionar um ativo real (resolve isso e o FP-APP-13 ao mesmo tempo). (3) Nunca deixar uma exception de parsing vazar como texto cru na UI — sempre ter um fallback humano.

**ATUALIZAÇÃO CRÍTICA — não é específico do ticker errado (screenshot 18):** com GOOG corretamente selecionado (o ativo que o usuário de fato possui), os MESMOS endpoints continuam retornando 500: `borrow?userAddress=DHG4p1...&ticker=G...` e `deposit?userAddress=DHG4p1...&ticker=G...`. Ou seja, o problema é sistemático nesses dois endpoints, não específico do default errado (XOM) — isso descarta a hipótese anterior de "causa raiz simples" e reabre a questão.

**Efeito colateral grave observado agora:** apesar do 500 persistente, a maior parte do painel funciona corretamente com dados computados localmente/client-side — Health Factor 6.23 (verde, saudável), holdings "0.009170935 GOOG / $3" corretos, LTV slider funcional. MAS um banner de aviso apareceu no topo: **"Borrowing $0.48 exceeds the $0.00 this position supports."** — uma mensagem que contradiz diretamente o resto da tela (Health Factor saudável, Max Borrow de $1.50 confirmado ontem no Portfolio). A hipótese mais provável: esse aviso específico É alimentado pela resposta (falha) desses endpoints 500 — quando a chamada de capacidade de borrow falha, o client parece "fail closed" tratando a capacidade como $0.00, só que gera um alerta incorreto e alarmante em vez de simplesmente não bloquear ou usar o valor já calculado localmente (que está certo, como prova o Health Factor).

**Severity:** Critical (reforçado) — é uma falha real e reproduzível de dois endpoints centrais (`/deposit`, `/borrow`) do fluxo mais importante do produto, com efeito direto e visível na UI (aviso incorreto e alarmante que pode fazer um usuário desistir de uma operação legítima e segura).

**Reprodução confirmada em múltiplos valores:** o mesmo aviso incorreto apareceu tanto pra $0.08 quanto pra $0.48 de tentativa de borrow — confirma que não é um caso de borda ligado a um valor específico, é o estado de falha dos endpoints (`/deposit`, `/borrow` retornando 500) se propagando pra qualquer tentativa de borrow, independente do amount. Reforça ainda mais que é um bug sistemático no fluxo, não um edge case isolado.

**CONFIRMAÇÃO FINAL:** com GOOG corretamente selecionado (não mais XOM por engano), Health Factor mostra "1.00 / 37.40" (saudável) e "Est. borrower cost/yr: $0.00" — mas o erro cru **"CollateralType: unexpected length 213 (expected 165, or 149 pre-migration)"** continua aparecendo, junto com "You receive $0.08" e o botão "Borrow $0.08 USDC" ativo. Isso prova definitivamente que o erro não depende do ticker selecionado (não é о bug do default XOM) — é uma falha real e persistente na camada de desserialização de conta, coexistindo com os 500s nos endpoints `/api/vault/deposit` e `/api/vault/borrow`. A causa mais provável agora: a conta de colateral do usuário tem 213 bytes, e nem o layout atual (165) nem o legado (149 pre-migration) batem — sugere um terceiro formato de conta não coberto pelo parser atual, possivelmente introduzido por uma feature mais recente (como o próprio Leverage) que não foi migrada corretamente nesse endpoint específico.

---

## FP-APP-13 — Painel de borrow abre com ativo errado por padrão ("Borrow against XOM" sem eu possuir XOM)

**What happened:** Ao entrar em `/borrow` com uma posição ativa apenas em GOOG (Position Value $2.99, Max. Borrow $1.50 — visíveis corretamente na tabela), o painel de ação à direita abriu pré-selecionado como "Borrow against **XOM**" — ativo que o usuário não possui nenhuma posição (linha mostra $0.00 / 0 shares). O painel também mostra "Health Factor: 1.00 / ∞" e "Borrow $0.00 USDC" nesse estado, que fazem sentido pra um colateral vazio, mas o ativo errado sendo pré-selecionado é confuso — o usuário logicamente esperaria que o painel abrisse já mostrando o ativo que ele de fato possui (GOOG), ou pelo menos um estado neutro/vazio, não um ativo aleatório sem posição.

**Severity:** Medium — não bloqueia o fluxo (dá pra clicar na linha do GOOG pra corrigir, presumivelmente), mas é uma primeira impressão confusa logo na entrada da tela mais importante do produto (borrow é o core value prop).

**TODO:** clicar na linha do GOOG na tabela e confirmar se o painel muda corretamente pra "Borrow against GOOG" com os valores certos.

---

## Seção 8 — Senior Analysis (rascunho, pra refinar antes da submissão final)

*O que tornaria o Spout imbatível — pontos estruturais que vão além dos findings individuais.*

**1. O funil de conversão quebra antes do usuário sequer decidir comprar.**
Não é um FP isolado — é um padrão. O banner "0% Interest. Always." contradiz a coluna "Borrow Cost" na mesma tela (FP-APP-1), e o Phantom bloqueia fisicamente a transação com "pode ser maliciosa" (FP-APP-5). Um usuário novo bate nesses dois sinais de alerta *antes* de qualquer decisão informada sobre o produto em si. Corrigir bugs de UX interna não resolve nada se o funil já quebrou na porta de entrada.

**2. O produto trata "erro de infraestrutura" e "erro de negócio" como a mesma coisa.**
O erro cru de desserialização de conta (FP-APP-12) e o aviso incorreto de "$0.00 de capacidade" nasceram do mesmo lugar: quando uma chamada de API falha (500), o client não distingue "não consegui buscar o dado" de "você não tem capacidade". Isso é sintoma de uma camada de error handling que não foi desenhada pensando em UX — só em happy path. Em fintech, estado de erro mal comunicado é uma falha de produto, não só de polish.

**3. O ambiente de teste (devnet) não se anuncia — e isso é um problema de confiança, não só de rótulo.**
FP-APP-14 não é sobre "esqueceram de colocar um badge". É sobre o fato de que um produto financeiro que lida com risco de capital real (no futuro, mainnet) treina o usuário, desde o beta, a não prestar atenção em qual rede está operando. Hábito formado agora é hábito carregado pro lançamento.

**4. A documentação e o produto contam duas histórias diferentes do mesmo mecanismo.**
Vimos isso repetidamente: yield alvo (7% vs 9%), liquidation fee (5% flat vs 8.8% no exemplo), "no losing your upside" vs o mecanismo real de assignment. Isoladamente cada um parece um typo. Juntos, formam um padrão: a camada de marketing/docs foi escrita antes (ou separadamente) da implementação final, e ninguém reconciliou as duas depois. Isso é resolvível com um processo, não com um dev fixando cada instância.

**5. O risco estrutural mais interessante do produto (RWA + mercado fechado) é o menos comunicado.**
Nada no produto avisa o que acontece com o Health Factor de um usuário se o preço de uma ação cair 15% num gap de abertura de segunda-feira, com o oracle rodando em frequência reduzida no fim de semana (achado da varredura de docs, `/oracles`). Esse é o risco mais nativo e diferenciador do Spout em relação a DeFi puro-cripto — e é justamente o que está menos explicado tanto nos docs quanto na UI.

**6. O produto ainda não decidiu se é "DeFi-native" ou "fintech regulada" na forma como comunica risco.**
Mistura linguagem de "0% interest, no margin calls" (tom fintech tradicional, tranquilizador) com mecânica real de covered calls e assignment (risco real, tipo opções). Protocolos DeFi-native maduros (Kamino, MarginFi) tendem a expor a mecânica de risco de forma mais crua e assumida — o usuário sabe que está em DeFi. O Spout tenta suavizar com linguagem fintech um produto que estruturalmente ainda carrega risco de derivativo.

**TODO:** validar ponto 6 com uma comparação direta (feature-by-feature) contra Kamino ou MarginFi antes de finalizar — ainda não fizemos essa comparação.

---

## Seção 6 — One-Sentence Test

*"Spout Finance deixa você tomar emprestado stablecoins a 0% de juros usando ações tokenizadas como colateral — financiado por covered calls sobre os mesmos ativos — mas ainda comunica risco de liquidação e leverage como se fosse fintech tradicional, não como o produto de derivativo que estruturalmente é."*

---

## FP-APP-4/6 — ESCALADO: FAQ público (schema.org) afirma KYC obrigatório, contradizendo diretamente a ausência observada

**What happened:** Inspecionando o HTML fonte de `spout.finance` (o conteúdo SSR/fallback servido antes da hidratação React — o que Google, crawlers e leitores de tela realmente indexam), há um bloco `FAQPage` em JSON-LD com respostas públicas e estruturadas. Duas delas são explícitas sobre compliance:

- *"How do I get started?"* → **"Connect a supported Solana wallet, complete a one-time KYC verification, and you can deposit equities..."**
- *"Is Spout compliant with US regulations?"* → **"Spout uses a regulated US broker-dealer for custody and options execution, enforces wallet-level KYC on all tokenized asset holders, and is structured to comply with applicable US securities laws."**

Isso é uma claim pública, indexável, estruturada especificamente para SEO/AI crawlers — não é um texto qualquer de marketing, é dado que o Google e assistentes de IA vão citar como fato sobre o produto.

**Por que isso eleva a severidade:** Anteriormente (FP-APP-4/6) tratamos a ausência de KYC como possivelmente aceitável — "é devnet, faz sentido pular verificação". Mas essa nova evidência muda o enquadramento: o site **afirma publicamente e estruturadamente** que KYC é enforced "wallet-level" em "all tokenized asset holders", sem nenhuma ressalva de "exceto no beta/devnet". Um usuário, auditor, ou parceiro institucional que ler essa FAQ (ou uma IA que a cite) forma uma crença factualmente incorreta sobre o estado atual do produto. Isso é diferente de "UX confusa" — é uma claim de compliance regulatório não verificável no produto real, o tipo de coisa que auditores de segurança/compliance marcam como Critical.

**Severity:** **Critical** (upgrade de Medium/High) — claim pública e estruturada de compliance regulatório (KYC "enforced... on all tokenized asset holders") não observável no fluxo real testado, em qualquer ambiente (nem sequer um aviso de "KYC required in production, skipped in beta" aparece).

**Suggested improvement:** Ou (a) implementar o gate de KYC mesmo em beta/devnet — pelo menos como fluxo simulado, pra validar que o enforcement funciona antes do mainnet, ou (b) adicionar uma ressalva explícita na FAQ pública: "KYC enforcement is active on mainnet; the current beta on devnet does not require it." Deixar a claim pública sem ressalva, enquanto o produto real não cumpre, é o tipo de gap que pode virar problema regulatório real, não só de UX.

---

## Nota — confirmações cruzadas úteis (HTML fonte)

- **Fee split lender/protocol confirmado:** FAQ pública diz "distributes 80% of the collected premium to lenders" — bate exatamente com os 20% de protocol fee already documentados em `/fee-structure` (100% - 20% = 80%). **Consistência real, sem contradição** — bom sinal, vale citar como "what works" (números batendo entre fontes diferentes).
- **Custo médio do borrower quantificado:** a FAQ diz "Historically this cost averages around 0.5% annualized across the portfolio" para o risco de assignment do borrower — isso pode ser a explicação real por trás do "Borrow Cost" da tabela de Trade (FP-APP-1/5), já que os valores observados (0.54%, 0.86%, 1.12%, 0.41%, 1.35%) giram em torno dessa média. Ainda não está explicado dentro do app via tooltip, mas o dado público existe — reforça a recomendação de trazer essa explicação pra dentro do produto, não só pra FAQ externa.
- **Inconsistência menor:** a FAQ pública promete "double-digit APY across 11 assets" pros lenders, mas a Senior tranche documentada é ~9% (não é double-digit). Só a Junior (~32%) cumpre a promessa. Achado Low — vale mencionar en passant.
- **200% de colateralização** mencionado na FAQ é consistente com LTV de 50% (50% LTV = 2x colateral = 200%) — apenas outra forma de expressar o mesmo número, não é inconsistência.

---

**FP-LANDING-1 — Flash de "2% Interest" durante carregamento antes de estabilizar em "0% Interest"**

**What happened:** Ao carregar a landing page, a seção "The Cost of Borrowing" exibe brevemente **"2% Interest — What the ultra-wealthy pay"** antes de estabilizar no valor final correto: **"0% Interest — What you pay with Spout"** (confirmado pelo usuário — o valor certo aparece, só havia um estado intermediário capturado no screenshot 19). Provavelmente é um contador animado (count-up/count-down) ou um placeholder de hidratação que renderiza um valor transitório antes do valor real assentar.

**Why it caused friction:** Mesmo sendo transitório, um flash de conteúdo incorreto na hero section — exatamente a claim central do produto — é um risco real: (1) qualquer print/screen-recording tirado nesse instante captura a versão errada (como aconteceu aqui), (2) para conexões lentas ou dispositivos mais fracos, esse estado intermediário pode durar tempo suficiente pra ser lido como afirmação real, (3) é o tipo de "flash of incorrect content" que ferramentas de acessibilidade e leitores de tela podem capturar literalmente antes da correção.

**Severity:** Medium — não é um bug de dado errado permanente (o valor final está correto: "0% Interest — What you pay with Spout"), mas é uma falha de polish em um lugar de alta visibilidade, com risco real de captura equivocada (como este próprio caso demonstra).

**Suggested improvement:** Se for um contador animado, iniciar a contagem a partir de um valor neutro (ex: "—%") em vez de mostrar "2%" como frame intermediário, ou aplicar um fade-in em vez de contagem visível. Se for estado de hidratação, garantir que o valor inicial no SSR já seja o correto (0%) antes do JS assumir.

---

**Findings totais: 26** (5 de documentação + 1 de landing page + 1 de mobile + 19 de app), divididos por severidade:
- **Critical: 8** — FP-APP-16 (venda marcada "Failed" mas executada de verdade — o mais grave do report), FP-MOBILE-1 (site não renderiza em mobile/Slow 4G), FP-APP-5 (bloqueio total no Phantom, buy e sell), FP-APP-12 (erro de desserialização + 500s sistemáticos), FP-APP-15 (Avg Cost/P&L errado no AAPL), FP-DOC-2 (liquidation fee inconsistente), FP-APP-4/6 (KYC publicamente prometido, ausente na prática), FP-APP-13/9 relacionados
- **High: 5** — FP-APP-1 (Borrow Cost vs "0% Always"), FP-APP-14 (devnet não sinalizado), FP-DOC-1 (yield inconsistente), FP-DOC-3 ("no losing upside" vs assignment)
- **Medium: 6** — FP-APP-13 (painel abre com ativo errado), FP-DOC-4/5 (LP reserve, circuit breaker sem threshold), FP-APP-7 (status "Executing" incorreto), FP-LANDING-1 (flash de "2%" no carregamento da hero)
- **Low: 4** — FP-APP-8 (arredondamento de shares, resolvido), FP-APP-10/11 (polling ineficiente, SEO)

**A tese central em 2 frases:** o produto tem um modelo financeiro genuinamente diferenciado (RWA como colateral, yield via covered calls) e uma engenharia de backend com pelo menos um bug real e sério (desserialização de conta), mas a camada de comunicação — marketing, docs, mensagens de erro — não foi reconciliada com a implementação em nenhum desses três lugares. O resultado é um produto que assusta usuários no primeiro contato (Phantom) e depois os confunde no meio do fluxo (mensagens contraditórias), mesmo quando o mecanismo subjacente funciona corretamente.

**As 3 correções que moveriam o ponteiro imediatamente:**
1. Resolver o flag de segurança do Phantom (contato direto com Phantom/Blowfish para allowlist) — é a barreira de conversão mais alta e mais barata de remover
2. Corrigir os endpoints `/api/vault/deposit` e `/api/vault/borrow` que retornam 500 (achado técnico #1) — afeta a confiabilidade percebida do core feature
3. Auditoria de reconciliação entre docs/marketing e a UI real (yield, liquidation fee, "no losing upside") — um passe de revisão resolveria 3-4 findings de uma vez

---

*Kamino escolhido como referência por ser o maior money market da Solana, estruturalmente similar (peer-to-pool, LTV + liquidation threshold + oracle-based), mas puro-cripto — o que evidencia onde o modelo RWA do Spout diverge.*

| Dimensão | Spout | Kamino Lend |
|---|---|---|
| **LTV máximo** | 50% flat, mesmo para todos os ativos | 70-80% típico, varia por ativo (blue-chip maior) |
| **Liquidation penalty** | Documentado como 5% flat, mas exemplo prático mostra 8.8% (ver FP-DOC-2) — **inconsistência não presente no Kamino** | 2-10%, **explicitamente variável e documentado como tal**: começa em 2% pra liquidadores rápidos, sobe até 10% conforme o LTV piora |
| **Tipo de liquidação** | Não documentado se é total ou parcial | **Soft liquidation**: fecha só a fração necessária da dívida (ex: 20%), suavizando o impacto pro borrower |
| **Interest rate model** | 0% fixo pro borrower, financiado externamente via covered calls | Flutuante, curva de utilização (kink) — modelo DeFi-native clássico, sem subsídio externo |
| **Fontes de oracle** | 1 oracle, frequência reduzida fora do horário de mercado dos EUA (ver `/oracles`) | **Pyth + Switchboard cross-referenciados**, redundância explícita |
| **Transparência de risco de liquidação** | Fee real depende do ativo mas isso não é comunicado antecipadamente ao usuário | Kamino documenta o range completo (2-10%) e a lógica de como o penalty sobe, publicamente, antes do usuário nem entrar na posição |

**O que isso revela (conecta com Senior Analysis, ponto 6):** o Kamino assume publicamente que liquidação é parte do jogo e documenta a mecânica com números concretos e variáveis por design. O Spout comunica uma fee "flat e baixa" (5%) que a prática desmente (8.8%) — não porque o modelo variável seja errado (faz sentido dado que ações voláteis carregam mais risco), mas porque a comunicação não acompanhou a implementação. Um usuário vindo de Kamino/MarginFi entraria no Spout esperando o mesmo nível de transparência quantitativa sobre risco de liquidação, e não encontraria.

**Nota sobre RWA vs cripto puro:** vale destacar que o LTV mais conservador do Spout (50% vs 70-80%) faz sentido dado o risco adicional de RWA (settlement T+1/T+2, mercado fechado nos fins de semana, gap risk) — esse é um ponto genuinamente bom de design, não uma crítica. A crítica é só sobre a comunicação da mecânica de liquidação, não sobre o parâmetro em si.

---

## O que funciona bem (pra Seção 5 — What Works, evitar report só-crítica)

- Loss waterfall bem desenhado e documentado com números concretos (Insurance Fund → Junior → Senior)
- `/liquidation-example` é detalhado e educativo, acima da média DeFi
- Earnings skip pra ações individuais (não abre calls durante earnings) — cuidado de risco que poucos protocolos pensam em implementar
- Registro FinCEN como MSB é verificável, reforça legitimidade regulatória
- Modelo de fees é simples de entender no nível macro (0% borrow, 20% protocol fee sobre premium)

---

## FP-APP-1 — "0% Interest. Always." vs coluna "Borrow Cost" com valores > 0% na própria tela de Trade

**What happened:** O banner no topo da tela de Trade diz, em destaque: "0% Interest on borrowing. Always, no matter the market conditions." A poucos centímetros abaixo, a tabela de ativos tem uma coluna chamada "Borrow Cost" com valores reais e não-zero por ativo: GS 0.86%/yr, XOM 0.80%/yr, MSTR 0.58%/yr, IBIT 0.55%/yr, GOOG 0.54%/yr, AAPL 0.37%/yr, PFE 0.07%/yr, GLD 0.06%/yr — e só NVDA, BSOL e SMCI mostram 0.00%/yr.

**Why it caused friction:** Isso contradiz diretamente a claim central do produto ("0% interest, always") na própria tela onde o usuário decide o que comprar. Se "Borrow Cost" for de fato uma taxa cobrada sobre o empréstimo, o headline de marketing do site inteiro está errado/enganoso. Se for outra coisa (ex: alguma métrica de risco disfarçada de "cost", ou o spread do covered call implícito no ativo), o nome do campo está péssimo e confunde qualquer usuário novo.

**Severity:** Critical — é a claim #1 do produto sendo aparentemente contradita na primeira tela que qualquer visitante vê, sem precisar nem conectar carteira.

**Suggested improvement:** Ou (a) renomear "Borrow Cost" pra algo que não confunda com "interest" (ex: "Assignment Probability Cost" ou "Est. Opportunity Cost"), com tooltip explicando o que realmente é, ou (b) se for de fato um custo de juros disfarçado, corrigir o banner.

**RESOLVIDO (parcialmente) via FP-APP-2:** o tooltip do Leverage (screenshot 02) diz literalmente: *"At 2.0x you put up half and borrow half at 0% interest. Higher leverage means more upside but also more **borrower cost**."* — ou seja, o próprio produto usa a expressão "borrower cost" na mesma frase em que reafirma "0% interest". Isso confirma que "Borrow Cost" na tabela de Trade não é juros no sentido tradicional, mas também confirma que EXISTE um custo real de tomar leverage, o que o banner "0% Interest. Always." simplesmente não comunica. A contradição não é um bug de copy isolado — é uma tensão consistente entre o headline de marketing e a explicação técnica real, repetida em pelo menos 2 lugares da UI (banner vs coluna, banner vs tooltip).

**Ainda em aberto:** a coluna "Borrow Cost" na tabela de Trade não tem tooltip próprio (não testado ainda) — não sabemos se ela é literalmente esse "borrower cost" do leverage, ou uma métrica diferente (ex: custo implícito de oportunidade do covered call daquele ativo). Precisa confirmar antes de fechar a análise.

---

## FP-APP-2 — "Leverage" slider (1.0x–2.0x) no fluxo de Buy não existe em nenhuma página da documentação

**What happened:** No painel de compra (BUY/SELL NVDA), há um slider de "Leverage" com marcações 1.0x / 1.25x / 1.5x / 2.0x, junto de um ícone de info (não clicado ainda). Nenhuma das 31 páginas de `/docs` menciona um mecanismo de "leverage" no momento da compra — os docs só descrevem alavancagem implícita via lock + borrow (50% LTV) depois que você já possui o spAsset.

**Why it caused friction:** É uma feature de risco real (alavancagem até 2x numa simples compra) sem nenhuma explicação prévia nos docs oficiais. Um usuário pode ativar 2.0x sem entender o mecanismo por trás (provavelmente compra a prazo/margin sintética via o próprio protocolo?) — isso é buy-side leverage, distinto do borrow-side LTV que os docs cobrem extensivamente.

**Severity:** Critical — risco financeiro não documentado é o tipo de achado que mais pesa em bounty de UX de lending/borrow.

**Suggested improvement:** Documentar o mecanismo de leverage no fluxo de Buy com a mesma profundidade que os docs dão ao borrow (Health Factor, liquidation, etc.), ou linkar o tooltip do "i" pra uma página de docs dedicada.

**TODO PRIORITÁRIO:** clicar no ícone "i" ao lado de "Leverage" e documentar exatamente o que aparece. Testar mover o slider pra 1.25x e ver se aparece algum aviso, health factor preview, ou mudança na tela.

---

## FP-APP-3 — Tab "Borrow" — REVISADO: não é bug, é empty state (rebaixado de Critical)

**What happened:** A página `/borrow` não está vazia por bug — é um empty state intencional: "Trade stocks, unlock 0% borrowing. Every $10,000 of eligible stock lets you borrow up to $5,000 USDC at 0% interest, without selling anything." + botão "Explore Stocks" que leva de volta pra `/trade` (screenshot 03).

**Why it's actually decent UX:** Confirma corretamente o LTV de 50% (bate com os docs: $5,000/$10,000 = 50%). Direciona o usuário sem colateral pro fluxo certo. Isso não é friction — é onboarding funcionando como deveria.

**Severity:** rebaixado para Low/Info — vale citar no report como exemplo de "what works" (empty states bem guiados), não como problema.

**Novo TODO:** comprar uma posição pequena em NVDA ou outro ativo primeiro, depois voltar em `/borrow` pra ver a tela real de lock+borrow, aí sim testar o fluxo documentado (Health Factor, LTV, etc.)

---

## FP-APP-4 — Nenhum passo de KYC em todo o fluxo de connect (contradiz /security-and-compliance)

**What happened:** Fluxo completo de connect documentado (screenshots 05-09): "Log in or sign up" (Privy, email ou wallet externa) → "Select your wallet" (Phantom/Solflare/Backpack/Jupiter/WalletConnect) → aprovação no Phantom → toast "Wallet verified for devnet — you can place test orders now." → painel de wallet mostra saldo devnet ($34.73 USDC de teste) já creditado, sem precisar pedir no faucet. Em nenhum momento desse fluxo apareceu qualquer tela de KYC, verificação de identidade, ou aceite de termos regulatórios.

**Why it caused friction / matters:** `/security-and-compliance` nos docs afirma que o protocolo usa "Token-2022 transfer hooks" pra enforcement de KYC diretamente on-chain. Se isso é real, o enforcement deveria bloquear ou pelo menos avisar ANTES de deixar o usuário "verificado pra devnet" e liberado pra ordens de teste. Pode ser que (a) o KYC só seja exigido no mainnet e o beta em devnet pula isso de propósito (razoável, mas não está explicado em lugar nenhum na UI), ou (b) o enforcement realmente só acontece no momento da transação (ex: tenta comprar e É bloqueado ali). Ambas as hipóteses precisam ser testadas antes de reportar como bug.

**Severity:** High — é uma claim de compliance central do produto (citada como parte do case regulatório inteiro) não observável no fluxo real testado até agora.

**TODO:** tentar uma compra real agora que a wallet está conectada e ver se o KYC aparece nesse momento (ex: modal de "verify your identity" ao clicar em confirmar ordem). Se a ordem passar sem qualquer verificação, documentar como achado forte pro Senior Analysis.

**Nota positiva:** a UX do connect em si é boa — devnet ativado automaticamente com fundos de teste já disponíveis, sem fricção de pedir manualmente no faucet pra começar (apesar do link do faucet também estar disponível caso precise de mais). Isso facilita MUITO o teste, vale citar como "what works".

---

- "Held 1:1 at Alpaca Securities · Reserves 100.2%" — bom, é a Proof of Reserve prometida nos docs, visível e com número real (100.2%, levemente acima de 1:1, provavelmente por arredondamento ou buffer).
- Tour guiado disponível ("Take a tour") — ainda não clicado; docs de UX-testing recomendam testar SEM o tour primeiro pra capturar fricção real, e DEPOIS comparar com o que o tour explica (ou deixa de explicar).
- "Market: Open" com relógio ao vivo (2:29:45) — bom sinal de transparência de horário de mercado, relevante pro ângulo de "o que acontece fora do horário" (FP-DOC riscos de gap).
- Nem todos os ativos têm Market Cap visível (BSOL, GLD, IBIT mostram "—") — possível gap de dado, checar se isso afeta cálculo de algo ou é só cosmético.
- **Market: Closed agora** (screenshot 04) — Trade screen continua totalmente navegável e o painel de Buy parece ativo mesmo com mercado fechado. TODO: tentar de fato submeter uma ordem de compra com o mercado fechado (sem wallet conectada ainda dá pra ver se o botão muda de "Connect Wallet" pra algo como "Market Closed" ou se fica igual). Relevante direto pro ângulo de risco de weekend/overnight gap (ver seção Financial Safety abaixo) — se a UI deixa parecer que dá pra operar normalmente fora do horário sem nenhum aviso, é friction.

---

## FP-APP-5 — Phantom exibe alerta vermelho "dApp pode ser maliciosa" + "domínio é novo" numa compra legítima

**What happened:** Ao confirmar a compra de GOOG, o Phantom Wallet exibiu dois avisos empilhados: um banner vermelho "Esta dApp pode ser maliciosa. Não prossiga, a menos que tenha a certeza de que é segura." e um banner amarelo "Este domínio é novo. Prossiga apenas se confiar neste site." O botão de confirmar aparece com o texto "Confirmar (não seguro)" em vez do padrão do Phantom.

**Why it caused friction:** Isso é o pior tipo de fricção possível num fluxo financeiro — o próprio software de segurança do usuário está dizendo "não confie nisso" no exato momento da conversão. Pra um usuário novo sem contexto prévio (ex: alguém clicando num anúncio ou vindo de conteúdo), esse alerta sozinho é suficiente pra abandonar a compra. É provavelmente um falso positivo da blocklist do Phantom por causa da idade do domínio (`beta.spout.finance` sendo novo/subdomínio de beta), mas o efeito pro usuário é o mesmo independente da causa.

**Severity:** Critical — é um dealbreaker de conversão que nenhum dos docs ou da própria UI do produto menciona ou prepara o usuário pra esperar.

**Suggested improvement:** (1) Submeter o domínio pra allowlist/verificação de segurança do Phantom, Solflare, Backpack etc antes do lançamento público (processo geralmente existe via formulário dos próprios wallets). (2) Enquanto isso não resolve, adicionar um aviso PRÓPRIO no fluxo de compra tipo "sua wallet pode mostrar um alerta de domínio novo — isso é esperado durante o beta, veja como verificar que é seguro" — assim o produto se antecipa ao medo em vez de deixar o usuário sozinho com um alerta vermelho.

**TODO:** testar se o mesmo alerta aparece com Solflare/Backpack (pode ser blocklist específica do Phantom) — se for só um wallet, é ainda mais fácil de reportar/resolver com eles diretamente.

## FP-APP-5 — ESCALADO: Phantom evolui de "aviso" pra "Pedido bloqueado" completo, confirmado em múltiplos ativos

**Atualização crítica:** o que começou como aviso vermelho (GOOG, screenshot 10) e depois um erro de simulação (BSOL, screenshot 13) agora escalou pra **bloqueio total** numa tentativa de compra de PFE (screenshot 14): tela cheia "Pedido bloqueado" / "Para sua segurança, a Phantom bloqueou este pedido." O único caminho pra prosseguir é o link de baixa visibilidade "Continuar na mesma (não seguro)" — a maioria dos usuários vai simplesmente fechar e desistir aqui.

**Por que isso muda a severidade e o enquadramento:** Já não é mais um problema isolado de um ativo (BSOL) — está acontecendo em GOOG, BSOL e PFE, ou seja, é a **dApp inteira** (`beta.spout.finance`) que está flagada na blocklist de segurança do Phantom, não uma transação ou ativo específico. Isso é o achado mais crítico e mais acionável do report inteiro: **qualquer usuário novo usando Phantom (a wallet Solana mais popular) que tente comprar qualquer ativo no beta vai bater nesse bloqueio total antes mesmo de completar a primeira compra.**

**Severity:** Critical, prioridade #1 do report — é um bloqueador de conversão de topo de funil, afeta 100% dos usuários de Phantom, e é resolvível rapidamente (processo de allowlist/reporte de falso positivo direto com a equipe de segurança do Phantom, normalmente via formulário ou contato direto).

**Dado novo, positivo, capturado nesse mesmo screenshot:** o painel de compra do PFE agora mostra um breakdown de custo que não tínhamos visto: "Amount / Stocks borrowed: $0.00 / 0", "Total amount / Stocks owned: $3.00 / 0.11", "**Est. borrower cost/yr: $0.00**" (em verde) e "Your cost today: $3.00". Isso é uma resposta parcial ao FP-APP-1 (Borrow Cost) — em Leverage 1.0x (sem alavancagem), o "Est. borrower cost/yr" é $0.00, o que sugere que o "Borrow Cost" da tabela só se aplica quando há de fato leverage/borrow envolvido. Falta ainda testar com leverage > 1.0x pra confirmar se o número sobe e bate com o "Borrow Cost" mostrado na tabela pra aquele ativo.

**Resposta do time Spout (Telegram, 09-10/09/2026):** "we're currently in beta and the product is still being tested, as the audit hasn't been completed yet. also, you don't have to interact with real money. all you need to do is request testnet USDC from the Solana Devnet faucet and interact with the product from there. so for now, it might look that way, but we're 100% legit and actively working toward the mainnet launch."

**Análise da resposta:** o time confirma que sabe do aviso e explica o contexto (beta pré-auditoria, sem risco de fundos reais). Isso é útil pra reduzir a preocupação de "é golpe?", mas **não resolve a causa raiz do FP-APP-5**: o bloqueio do Phantom não é sobre a legitimidade do projeto — é sobre a idade/reputação do domínio na blocklist de segurança da wallet, algo que normalmente se resolve com um processo de allowlist/reporte direto com a equipe do Phantom (ou Blowfish/outras firms de detecção que os wallets usam), independente de auditoria de smart contract estar pronta ou não. Vale reforçar isso no report como um follow-up específico: "isso é resolvível hoje, antes mesmo do audit terminar, e resolve uma perda de conversão real."

**Valor pro report:** essa troca em si é uma boa prova de comportamento proativo (Seção de metodologia/adversarial self-review) — reportei o achado ao sponsor antes de submeter, tenho o timestamp e a resposta documentados.

---

**What happened:** A compra de GOOG completou de ponta a ponta — Phantom confirm → "Order placed" → "Your purchase has been confirmed" — sem nenhuma tela de KYC/verificação de identidade em nenhum ponto do fluxo, incluindo o momento exato da transação on-chain.

**Why it matters:** Resolve a dúvida aberta do FP-APP-4: pelo menos em devnet, não há enforcement de KYC visível em lugar nenhum do fluxo de compra, apesar de `/security-and-compliance` descrever Token-2022 transfer hooks pra isso. Reforça a hipótese de que o gating de compliance só existe no mainnet e o beta pula isso de propósito, mas isso não está comunicado em nenhum lugar da UI.

**Severity:** Medium (rebaixado de High já que provavelmente é comportamento esperado de devnet, mas ainda vale reportar a falta de comunicação disso)

---

## FP-APP-7 — Mensagens inconsistentes sobre status da ordem com mercado fechado

**What happened:** O modal de confirmação diz claramente "Market is closed. Your order fills at 9:30 AM ET, 10 Sep" (screenshot 11) — ótima transparência. Mas a tabela de "Open Orders" logo em seguida mostra Status = **"Executing"** (screenshot 12), o que sugere que algo está acontecendo agora, não que está numa fila esperando a abertura do mercado.

**Why it caused friction:** Pequena mas real — "Executing" e "fills at 9:30 AM ET amanhã" comunicam coisas diferentes. Um usuário que só olha a tabela de Open Orders (sem lembrar do modal) pode achar que a ordem está sendo processada agora e ficar confuso por que não reflete no saldo.

**Severity:** Low — quick win fácil de corrigir (trocar o label pra "Queued" ou "Pending market open").

**Suggested improvement:** Status label deveria mudar pra "Queued" ou "Scheduled" quando o mercado está fechado, reservando "Executing" pra quando a ordem já está de fato em processamento.

---

## FP-APP-8 — Total pago não bate exatamente com o valor de shares exibido (arredondamento)

**What happened:** "You own 0.01 GOOG" / "Bought at $328.39" / "Total paid $3.00". Mas 0.01 × $328.39 = $3.28, não $3.00. Provavelmente o usuário digitou "$3" como Amount e o app calculou as shares fracionárias reais (≈0.00913) mas exibiu arredondado pra "0.01" na tela de confirmação.

**Why it caused friction:** É um problema de precisão de exibição, não de cálculo (o total pago de $3.00 está provavelmente certo, é a contagem de shares que está arredondada de forma enganosa). Um usuário que confia no "0.01 GOOG" mostrado e depois vai vender pode se surpreender que o saldo real é menor.

**Severity:** Low/Medium — não é um bug financeiro real, é uma questão de exibição, mas em produto financeiro qualquer imprecisão de número gera desconfiança.

**RESOLVIDO/ATUALIZADO — Portfolio confirma que o dado real está correto:** a tela de Portfolio (10/09/2026, mercado aberto) mostra a posição com precisão completa: **0.009171 GOOG**, preço $326.14, valor de mercado $2.99. Isso confirma que o cálculo sempre esteve certo — o problema era só a exibição arredondada no modal de confirmação da compra ("0.01 GOOG"). Na tabela de Trade, o "Shares owned" agora mostra "< 0.01" pra essa posição, o que é uma exibição correta e honesta (evita a falsa precisão do "0.01" exato). **Rebaixado definitivamente pra Low** — é só o modal de confirmação pós-compra que precisa mais casas decimais; o resto do produto já trata isso bem.

---

## FP-APP-9 — BSOL: Phantom falha em simular a transação ("Não foi possível simular os resultados desta solicitação")

**What happened:** Ao tentar comprar BSOL (Bitwise Solana Staking ETF), o Phantom exibiu os mesmos avisos de "dApp maliciosa" / "domínio novo" do FP-APP-5, **mais** um terceiro alerta vermelho novo e mais grave: "Não foi possível simular os resultados desta solicitação." O botão de confirmar continua disponível como "Confirmar (não seguro)" — ainda não confirmado, aguardando.

**Why it's different/worse than FP-APP-5:** O aviso de "domínio novo" é reputacional (idade do domínio na blocklist). Já "não foi possível simular" é um sinal técnico — o Phantom tenta rodar a transação num ambiente de simulação antes de assinar, pra prever o resultado (que tokens saem, que tokens entram) e mostrar pro usuário. Quando essa simulação falha, geralmente indica uma de duas coisas: (a) a transação vai reverter/falhar on-chain mesmo, ou (b) a instrução usa algo que o simulador não processa bem (ex: CPI complexo, dependência de estado que só existe no momento exato da execução). Ambos os cenários merecem investigação antes de confirmar.

**Severity:** Critical — se a causa for (a), é um bug real que vai desperdiçar gas do usuário numa transação fadada a falhar. Se for (b), ainda assim é uma fricção grave: o usuário perde justamente a preview que deveria dar confiança pra assinar.

**TODO PRIORITÁRIO:** decisão a tomar quando formos confirmar — (1) se confirmar e a tx passar on-chain com sucesso, documentar que foi falso alarme de simulação (mas ainda reportável como friction de UX). (2) se falhar on-chain, é uma prova concreta de bug real, com hash de erro pra documentar. Comparar também se esse erro é específico de BSOL (talvez por ser um ETF/staking token com lógica diferente de uma ação comum) ou se acontece com qualquer ativo — já vimos GOOG confirmar sem esse erro específico de simulação (só os avisos reputacionais), então é bem possível que seja específico do BSOL.

**Nota:** ainda não confirmada — aguardando decisão de prosseguir ou cancelar antes de registrar o resultado final.

---

- Transparência sobre mercado fechado: avisa claramente ANTES de confirmar que a ordem só executa na próxima abertura (9:30 AM ET) — melhor que muitos produtos tradicionais de corretora que escondem isso.
- Fluxo de compra é rápido: da tela de Trade até "purchase confirmed" em poucos cliques, sem redirecionamentos desnecessários.
- Toast de "Order placed — tx [hash]" com link implícito pro explorer é uma boa prática de transparência on-chain que os docs prometem e a UI de fato entrega.

---

- [x] Landing page → tela de Trade carrega direto, SEM exigir connect pra ver preços/dados — bom pra fricção zero de descoberta
- [ ] Wallet connect: o que estava claro / confuso? (ainda não clicado)
- [x] Onboarding: KYC não apareceu em nenhum momento do connect (ver FP-APP-4) — verificar se aparece na hora da ordem
- [x] Primeira tela pós-connect: painel de wallet mostra saldo devnet, é claro e direto — bom onboarding de teste
- [x] Primeira hesitação real capturada: "Borrow Cost" != "0% interest" (ver FP-APP-1 acima)
- [x] Segunda hesitação real capturada: "Leverage" slider sem explicação (ver FP-APP-2 acima)
- [ ] Screenshot desta tela salvo em assets/screenshots/

---

## Confirmação: ordem GOOG preencheu corretamente (10/09/2026, mercado aberto)

- Order placed ontem (mercado fechado) → hoje 9:30 AM ET a ordem executou normalmente, como prometido no modal ("Market is closed. Your order fills at 9:30 AM ET, 10 Sep") — **promessa cumprida, sem surpresas**. Bom ponto pra "What Works": a comunicação de fill futuro foi precisa.
- **Portfolio Overview:** Total Equity $2.99, Total Borrowed $0.00, Net Worth $2.99, **Available Borrowing Power $1.50**
- Confirma LTV de 50%: $2.99 × 50% = $1.495 ≈ $1.50 — bate exatamente com os docs (`/how-borrowing-works`, `/health-factor`). Ótimo sinal de consistência entre documentação e implementação real.
- "My Holdings" mostra a posição com status "Active" e precisão completa nas shares (0.009171) — ver atualização do FP-APP-8 acima.
- "Positions" (parte de baixo, referente a posições de borrow) mostra "No Positions — Deposit collateral to open a position" — próximo passo natural agora: ir em `/borrow`, essa posição de GOOG já deveria aparecer como colateral disponível.

---

## Monitoramento da posição GOOG ao longo do tempo

**Dia 1 (10/09, fill):** 0.009171 GOOG @ ~$326.14, valor $2.99, LTV disponível $1.50
**Dia 2 (12/09, sexta à noite):** mesma posição (0.009171 GOOG), preço subiu pra $335.38, valor $3.08, **Unrealized P&L: +$0.09 (+2.87%)**, Available Borrowing Power subiu proporcionalmente pra $1.54 (50% de $3.08)

**Observação:** o "Available Borrowing Power" recalculou corretamente e em tempo real conforme o valor de mercado da posição mudou — confirma que o LTV de 50% é dinâmico (recalculado sobre o valor atual, não travado no valor de entrada), consistente com os docs de Health Factor. Bom sinal de correção matemática do sistema, mesmo com os bugs de API já documentados (FP-APP-12) — sugere que os bugs são na camada de exibição/comunicação de erro, não no cálculo core de portfólio.

**Abas exploradas (Dia 2):**
- **Transaction History**: registro limpo e completo — "Yesterday / 10:30 AM / Bought GOOG / 0.009171 shares at $326.03 / +0.009171 GOOG / $2.99 / Fees: -- / Completed". Tem botão "Export CSV", bom sinal de produto pensando em auditoria/contabilidade do usuário.
- **Activity**: "No Activity — Your open orders will appear here", empty state limpo, consistente com o padrão bom já visto em `/borrow`. Correto estar vazio (nunca confirmamos as tentativas de borrow de $0.08/$0.48).
- **Nota menor de precisão:** o preço de compra aparece como $326.03 aqui, mas telas anteriores (modal de confirmação, portfolio) mostraram $326.14 e $326.22 em momentos diferentes — provavelmente só reflete o preço exato no timestamp de cada consulta (mercado se moveu entre a compra e as visualizações), não é um bug, mas vale mencionar como nota de precisão no report caso o preço final registrado on-chain divirja do que aparece na tabela.

---

## Sessão de diversificação — 5 novas posições abertas com mercado fechado (11/09/2026)

Ordens colocadas propositalmente com mercado fechado, pra (a) reforçar o FP-APP-7 com mais amostras e (b) diversificar volatilidade pro monitoramento da semana:

| Asset | Amount | Status exibido | Placed |
|---|---|---|---|
| BSOL | $5.00 | Executing | 11 de set. |
| MSTR | $5.00 | Executing | 11 de set. |
| AAPL | $5.00 | Executing | 11 de set. |
| PFE | $5.00 | Executing | 11 de set. |
| GS | $3.00 | Executing | 11 de set. |

**Confirma FP-APP-7 em escala:** todas as 5 ordens mostram "Executing" (amarelo) mesmo com o mercado fechado — não é caso isolado do GOOG, é o comportamento padrão do sistema pra qualquer ordem colocada fora do horário. Reforça a recomendação: trocar pra "Queued"/"Pending Market Open" quando `Market: Closed`.

**Boa cobertura de volatilidade pro monitoramento da semana:** MSTR e BSOL (mais voláteis, cripto-adjacentes) vs PFE/AAPL/GS (mais estáveis) — vai permitir comparar Borrow Cost real por volatilidade quando as posições preencherem na segunda-feira de manhã (9:30 AM ET) e o Health Factor de cada uma ao longo da semana.

---

## Inteligência competitiva — thread de outro tester (@blessedboy32, 11/09)

**Confirmação independente forte:** ele também encontrou "CollateralType: unexpected length 213..." no fluxo de borrow do NVDA, e descreve o mesmo sintoma que vimos: "UI still shows a max borrow amount. Click it and it says the position supports $0.00." — isso é uma **segunda fonte independente confirmando o FP-APP-12**, reforça muito a credibilidade desse achado no report final (posso citar como "corroborated by other independent testers in the same beta cohort").

**3 bugs novos, ainda não testados por nós — TODO validar:**
1. **Modal de confirmação de venda reusa o template de compra**: "Sell confirmation modal still says 'Your purchase has been confirmed'" — bug de copy/template não trocado entre fluxos de buy/sell.
2. **Campos em branco no Portfolio**: "Portfolio shows Total Equity but Net Worth + Available Borrowing Power stay blank" — diferente do que vimos no nosso teste (onde esses campos apareceram corretamente), pode ser um estado específico (talvez após uma venda parcial, ou outro ativo).
3. **"Max sell" deixa poeira residual**: "Max sell leaves dust instead of fully closing the position" — sugere que o botão "Max" no Sell não vende 100% da posição, sobra uma fração residual.

**Ângulo de UX friction dele, vale considerar:**
- "Heavy gating (email + passcode) slows real testing" — friction de onboarding que talvez não tenhamos sentido tanto (usamos Phantom direto, sem o fluxo de email do Privy). Vale testar o fluxo de email+passcode também, se ainda não tivermos feito.
- "High cognitive load — locking into weekly options cycles, health factor, potential assignment" — visão qualitativa que reforça nosso Senior Analysis (ponto 6, sobre a mistura de linguagem fintech tranquilizadora com mecânica real de derivativo).

**Como usar isso no report:** não precisa (nem deve) citar o tester nominalmente — mas vale registrar que o achado técnico #1 foi corroborado externamente, e testar os 3 bugs novos (venda parcial/total de uma posição pequena, ex: vender uma fração do GOOG) antes de fechar o report, já que temos posição ativa suficiente pra reproduzir.

---

## FP-APP-15 — CRÍTICO: Avg Cost do AAPL incorreto, causando erro massivo de P&L exibido

**What happened:** Segunda-feira, mercado aberto, as 5 posições novas preencheram (screenshot 36). Validando a matemática de cada linha (shares × avg cost deveria bater com o valor da ordem original, ~$5 ou $3):

| Asset | Shares | Avg Cost | Cost Basis Calculado | Valor da ordem | Bate? |
|---|---|---|---|---|---|
| MSTR | 0.038121 | $130.90 | $4.99 | $5.00 | ✅ |
| BSOL | 0.35822 | $13.93 | $4.99 | $5.00 | ✅ |
| PFE | 0.177013 | $28.19 | $4.99 | $5.00 | ✅ |
| GOOG | 0.009171 | $326.03 | $2.99 | $2.99 | ✅ |
| GS | 0.00296 | $1,010.00 | $2.99 | $3.00 | ✅ |
| **AAPL** | **0.014912** | **$334.62** | **$4.99** | **$5.00** | ✅ (a ordem em si bate) |

A ordem do AAPL bate matematicamente com a intenção original ($5 → 0.014912 shares a $334.62). **O problema é que $334.62 não é um preço remotamente próximo do AAPL real no momento da compra** — o preço atual exibido na mesma linha é $228.50, uma diferença de 46%. Se o Avg Cost estivesse certo (perto de $228), o P&L seria próximo de zero (mercado mal se moveu desde a ordem). Mas com Avg Cost em $334.62, a perda real calculável é **-$1.58 (-31.7%)** — só que a tela mostra **"$-0.02 (-0.50%)"**, um número completamente diferente e igualmente errado (nem bate com o Avg Cost errado, nem com um Avg Cost correto).

**Why this is critical:** É um erro duplo — (1) o Avg Cost gravado pra AAPL está incorreto por uma margem enorme (o dado de preço de execução usado pareceu ter vindo de outro ativo/período, "$334" não é preço plausível recente do AAPL), e (2) o P&L exibido nem sequer é consistente com o Avg Cost errado — é um terceiro número, sugerindo que o cálculo de P&L na tela usa uma fonte de dado diferente da que populou o Avg Cost. Isso é o tipo de bug que, em produção real com dinheiro de verdade, mostraria ao usuário uma perda/ganho completamente fictício.

**Severity:** Critical — integridade de dado financeiro é a categoria mais sensível possível num produto desse tipo; um P&L exibido incorretamente pode literalmente levar a decisões de venda/hold erradas.

**TODO:** verificar se esse erro é específico do AAPL (talvez colisão de preço com outro ativo no backend, dado que $334 não corresponde a nada óbvio) ou se acontece com qualquer ativo em certas condições. Comparar com o hash da transação on-chain do AAPL se possível, pra confirmar se o preço de execução real (on-chain) bate com $228 (correto) ou $334 (o que está sendo exibido).

---

## FP-APP-16 — CRÍTICO: Ordens de venda marcadas como "Failed" mas EXECUTADAS de verdade (shares realmente debitadas)

**What happened:** Duas ordens de venda (PFE e GOOG, 15/09) aparecem na tabela "Open Orders" com status **"Failed"**, cada uma com um detalhe de execução parcial estranho (ex: "Failed 0.036519942 @ $27.43"). Comparando os holdings antes e depois dessas ordens:

| Asset | Shares antes | Shares depois | Diferença | Bate com o "Failed" mostrado? |
|---|---|---|---|---|
| GOOG | 0.009171 | 0.003316 | 0.005855 | ✅ bate com "0.005854972" |
| PFE | 0.177013 | 0.140493 | 0.036520 | ✅ bate com "0.036519942" |

**As duas vendas realmente aconteceram — as shares foram debitadas exatamente na quantidade que a ordem "Failed" registra — mas o status exibido diz que falhou.**

**Why this is the most severe finding of the test:** isso é pior que os 500s ou o erro de deserialização, porque aqueles pelo menos comunicavam claramente que algo deu errado. Aqui, o sistema **executa a transação e mente sobre o resultado**. Consequências reais: (1) um usuário vendo "Failed" pode tentar vender de novo, potencialmente vendendo mais do que pretendia; (2) o saldo real do usuário diverge do que ele acredita ter, sem nenhum aviso; (3) num cenário de produção com dinheiro real, isso é o tipo de bug que gera disputas de suporte e perda de confiança irreversível.

**Severity:** Critical — o mais alto do report inteiro. Provável causa raiz: a mesma classe de problema do FP-APP-12 (erro de comunicação entre a camada que processa a transação on-chain e a camada que atualiza o status da UI) — a transação provavelmente teve sucesso na blockchain, mas a chamada de confirmação/callback que deveria marcar "Completed" falhou ou teve timeout, deixando o status "congelado" em "Failed" por padrão de erro.

**ATUALIZAÇÃO CRÍTICA — confirmado: USDC NÃO foi creditado (pior do que parecia inicialmente):** verificando o saldo de USDC da wallet antes e depois das duas ordens "Failed": permanece em **$8.73** nas duas leituras, sem nenhum aumento. Se as vendas tivessem de fato liquidado (shares → USDC), o saldo deveria ter subido em torno de $3 (≈$0.99 do PFE + ≈$2.00 do GOOG, baseado nos valores de mercado no momento). Isso muda a gravidade do achado: não é apenas "status incorreto numa transação que teve sucesso" — é **shares debitadas do holdings sem o USDC correspondente aparecer na wallet**. Duas hipóteses, ambas graves:
1. A perna de venda do swap (shares → SOL/USDC) executou on-chain (por isso os holdings caíram), mas a perna de crédito falhou de verdade, deixando o usuário com posição reduzida e sem receber nada em troca — perda real de valor.
2. O número de shares exibido no Holdings é decrementado de forma otimista/local assim que a ordem é enviada, antes de confirmação on-chain — e nesse caso a UI está mentindo sobre o saldo real na direção oposta (mostra menos do que você realmente tem), o que também é grave, só que num sentido diferente (sugere vender de novo o que já foi "gasto" só na tela).

**RESOLVIDO via Solscan devnet — causa raiz confirmada, reclassificação necessária:** inspecionando o histórico on-chain da wallet (`solscan.io/account/DHG4p1...RnNV?cluster=devnet`), as duas ordens de venda aparecem como instrução `placeSellOrder` (timestamps batendo exatamente com os horários das ordens "Failed"), mas **não existe nenhuma instrução `fulfillSellOrder` ou equivalente de liquidação/settlement subsequente** nas transações mais recentes da wallet. Ou seja: a ordem foi colocada on-chain de verdade (por isso os holdings caem — as shares ficam reservadas/travadas na ordem pendente), mas o passo de *fulfillment* (que devolveria USDC e finalizaria a venda) nunca aconteceu.

**Reclassificação:** isso não é "shares perdidas sem contrapartida" (hipótese 1) — é uma ordem que ficou **travada em estado pendente/não-preenchido, e a UI rotula incorretamente esse estado como "Failed"** (terminal, implica que nada aconteceu) quando deveria mostrar algo como "Pending Fulfillment" ou "Awaiting Settlement". O botão "Cancel" visível na tabela de Open Orders provavelmente devolveria as shares — mas isso não foi testado ainda.

**Achado bônus relevante (conecta com FP-APP-4/6):** no histórico de compras mais antigas da mesma wallet, aparece um padrão repetido de `adminThaw` seguido de `fulfillBuyOrderFreezeGated` — essa é a implementação real do mecanismo de freeze/thaw do Token-2022 que os docs (`/security-and-compliance`) descrevem como enforcement de KYC. Confirma que a arquitetura de compliance existe on-chain (tokens nascem "congelados" e um `adminThaw` os libera), mas no devnet esse thaw parece acontecer automaticamente/sem gate visível de KYC real — consistente com nossa observação de que nenhum passo de KYC apareceu na UI.

**Severity revisada:** Critical se mantém — não por perda de fundos, mas porque (a) o rótulo "Failed" é factualmente incorreto e engana o usuário sobre o estado real da sua posição, e (b) shares ficam efetivamente indisponíveis (nem na carteira pra vender de novo, nem convertidas em USDC) até o usuário descobrir que precisa cancelar manualmente — se é que o cancel resolve.

**TODO:** testar clicar "Cancel" numa dessas ordens travadas e confirmar se as shares voltam pro holdings.

---

## FP-APP-17 — Modal de confirmação de venda reusa o texto "Your purchase has been confirmed" (bug de template, confirmado independentemente)

**What happened:** Ao confirmar uma venda de GOOG e de PFE, o modal de sucesso mostra **"Your purchase has been confirmed"** — mesmo sendo uma venda, não uma compra (screenshots 44, 45; mostrado no popup mesmo após clicar SELL). Isso confirma exatamente o que o outro tester independente (@blessedboy32) reportou publicamente: "Sell confirmation modal still says 'Your purchase has been confirmed' (reused buy template)".

**Severity:** Medium/High — não é financeiramente perigoso sozinho, mas é confuso (usuário vendendo pode se assustar achando que comprou por engano) e, combinado com o FP-APP-16 acima, adiciona mais uma camada de sinais contraditórios exatamente no fluxo mais sensível do produto (mover dinheiro).

**Suggested improvement:** Criar um modal de confirmação de venda próprio ("Your sale has been confirmed" + ícone/copy adequados), não reaproveitar o de compra.

---

## FP-APP-18 — Phantom bloqueia SELL também, não só BUY

**What happened:** O mesmo alerta "Pedido bloqueado" do Phantom (FP-APP-5) apareceu numa tentativa de **venda** de GOOG (screenshot 43), confirmando que o bloqueio de segurança cobre qualquer transação da dApp, não só compras. Isso amplia o escopo do achado #1 do report — o problema não é específico do fluxo de compra, é qualquer interação com o contrato.

**Severity:** já coberto pelo Critical do FP-APP-5, essa é só uma confirmação de escopo mais amplo — vale mencionar no texto final.

---

## FP-APP-19 — Sem filtro de "só meus ativos" na tabela de Trade/Sell

**What happened:** O dropdown "All Types" só filtra por Stocks/ETFs (screenshot 40), não existe opção de "My Holdings" ou "Owned only" pra ver rapidamente só os ativos que o usuário já possui. Com portfólio de 6+ ativos, achar os que você tem exige rolar a tabela inteira e checar a coluna "Shares owned" manualmente.

**Severity:** Medium — friction real de usabilidade, mais perceptível quanto mais ativos o usuário acumula (como no nosso caso, após a diversificação da semana).

**Suggested improvement:** Adicionar "My Holdings" como opção no dropdown "All Types", ou um toggle rápido acima da tabela.

---

## Nota — Earn confirmado como não disponível ainda (não é falha nossa de teste)

A aba Earn mostra "Earn is coming soon. Lending vaults are on the way. In the meantime, you can trade tokenized stocks or borrow against the ones you already hold, at 0% interest." (screenshot 46) — confirma o que o brief da bounty já avisava (lending market "rolling out soon"). Não testamos o lado lender porque **não existe ainda em beta**, não por lacuna nossa. Vale mencionar no report pra deixar claro que a cobertura de "DeFi/tokenization analysis" ficou limitada ao lado borrower por escopo do produto, não por omissão do teste.

---

## FP-MOBILE-1 — CRÍTICO: página falha completamente em mobile com throttling Slow 4G (NO_FCP)

**What happened:** Rodando PageSpeed Insights em modo Mobile (Moto G Power emulado, Slow 4G throttling) na home `beta.spout.finance` (screenshot/doc 9, capturado 15/09), **a página não renderizou nenhum conteúdo**: "The page did not paint any content... (NO_FCP)" — erro em TODAS as métricas (FCP, LCP, TBT, CLS, Speed Index) e em praticamente toda a auditoria de Accessibility, Best Practices e SEO ("Error!" generalizado, porque o Lighthouse não conseguiu nem carregar a página pra analisar).

**Why this is critical:** O Lighthouse desktop rodado anteriormente (FP-APP-11) deu 100/100/100/91 — excelente. Esse resultado mobile é o oposto completo: falha total de renderização sob condições de rede realistas (Slow 4G é o cenário padrão de teste do Google pra simular conexão móvel comum, não um caso extremo). Isso sugere um problema sério de performance/bundle size que só se manifesta sob banda limitada — coerente com o padrão de polling ineficiente já documentado (FP-APP-10) e o bundle JS fortemente fragmentado em dezenas de chunks (visto no Sources tab, FP-APP-11) que pode estar competindo por banda limitada de forma severa.

**Severity:** Critical — significativa parcela do tráfego real de qualquer produto (inclusive DeFi) vem de mobile, e "a página não carrega" é o pior resultado possível, pior que qualquer bug individual de UX.

**TODO:** reproduzir manualmente num dispositivo mobile real com throttling de rede (Chrome DevTools mobile emulation + Network throttling) pra confirmar visualmente o que acontece — tela branca? Loading infinito? — e capturar screenshot/vídeo como evidência mais direta que só o relatório do Lighthouse.

---

**Checkpoint final (17/09):** Portfolio $21.26, Unrealized P&L -$0.05 (-0.18%), Available Borrowing Power $10.63, Total Borrowed $0.00. Gráfico do 1W mostra pico de $25.94 em 17/09 18h, com leve queda até o momento da leitura — variação real capturada ao longo da semana toda, do fill inicial até agora. **Monitoramento de posição encerrado aqui — segue pra montagem do report final.**

---

## FP-APP-20 — CRÍTICO: Portfolio inteiro zera ("No Holdings", "No Metrics") apesar das posições existirem

**What happened:** Numa sessão posterior (19/09), a tela de Portfolio mostra "No Holdings — Trade tokenized stocks to build your portfolio" e "No Metrics — Build your portfolio to see your total equity, borrowed amount, and net worth" — como se a conta nunca tivesse tido nenhuma posição. Só que o gráfico de "Portfolio Overview" no mesmo instante ainda mostra o pico histórico "$27.35, Sep 19, 12:00 AM", provando que os dados existiram e foram registrados. As 6 posições diversificadas (GOOG, MSTR, BSOL, PFE, GS, AAPL) simplesmente não aparecem mais.

**Why this is critical:** É o mesmo padrão de falha já visto em FP-APP-12/16 (API de dados falhando e a UI caindo num empty state incorreto, em vez de mostrar erro ou dado cacheado), só que agora atinge a visão mais importante do produto pro usuário — "quanto eu tenho". Diferente de um erro pontual numa tela secundária, esse esconde o portfólio inteiro.

**Severity:** Critical — não há indicação de liquidação real (Total Borrowed sempre foi $0, sem risco de margin call), então é quase certamente um bug de exibição/fetch, não perda real — mas a experiência pro usuário é indistinguível de "meu dinheiro sumiu" até prova em contrário.

---

## FP-APP-21 — Labels de ativo corrompidos na tabela de Open Orders (regressão)

**What happened:** As mesmas duas ordens travadas do FP-APP-16 (PFE e GOOG, vistas nos dias anteriores como "PFECLzi…UbBA" e "GOOG7zo3…H3nV" — ticker + endereço) agora aparecem como **"EG3r…CLzi…UbBA"** e **"6a2y…7zo3…H3nV"** — o ticker sumiu completamente, restando só fragmentos de endereço on-chain, ilegíveis pro usuário.

**Why this matters:** É uma regressão — a mesma tela piorou ao longo do teste, não melhorou. Combinado com o FP-APP-20, sugere uma falha mais ampla na camada que resolve metadata de ativos (nome/ticker) a partir do endereço on-chain, não só nessas duas telas específicas.

**Severity:** Medium/High — não esconde dinheiro, mas destrói a legibilidade de uma tela que já estava com status incorreto ("Failed").

---

## Log de transações on-chain (preencher conforme testar)

| Action | Amount | Asset | Transaction | Data |
|---|---|---|---|---|
| Buy | $3.00 | GOOG (0.01 shares, rounded) | 4ZReviwT... (confirmar hash completo no explorer) | 09/09/2026, market closed, fills 9:30 AM ET 10/09 |

---

## Ideias soltas / observações do dia 0

-
