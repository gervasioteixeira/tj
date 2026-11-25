# Calculadora de Selos de Produtividade — TJPB (Res. 49/2026)

## O que é

Simulador **não-oficial**, em página HTML única, do Selo de Eficiência do TJPB
instituído pela **Resolução TJPB nº 49, de 07/05/2026** (processo SEI
007626‑95.2026.8.15). O usuário preenche seus indicadores e a página calcula,
em tempo real, a pontuação relativa estimada, qual selo (Prata/Ouro/Diamante/
Excelência) seria alcançado e sugestões do que melhorar primeiro.

Hospedado (GitHub Pages) em **https://gervasioteixeira.github.io/selo2026/**,
repositório `gervasioteixeira/selo2026`, arquivo `index.html`. O rodapé público
indica "ferramenta independente, sem vínculo oficial" — não reintroduzir nome
ou e-mail pessoal no rodapé.

Público-alvo: servidores do TJPB (varas/juizados/turmas recursais) que querem
entender como o selo é calculado e simular cenários. Não substitui a apuração
oficial da Gerência de Planejamento.

## Arquitetura

- **Arquivo único** `index.html` — HTML + CSS (`<style>`) + JS (`<script>`)
  inline, sem build, sem dependências, sem rede. Roda via `file://` ou
  servidor estático (ex.: Laragon).
- JS puro, tudo em um `<script>` no fim do `<body>`.
- **Persistência**: `localStorage` (nada vai para servidor), salvamento com
  debounce de 400ms (`scheduleSave`/`saveState`/`loadState`).
- Dark mode via `prefers-color-scheme`; responsivo (breakpoints 980px e 600px).
- Sem testes automatizados. Validação usada até aqui: checagem de balanceamento
  de tags (`html.parser`), `node --check` no `<script>` extraído e Playwright
  (Chromium headless) para screenshots mobile/desktop, claro/escuro.

## Base legal / fontes

1. **Resolução TJPB nº 49/2026** — Art. 3º (faixas dos selos), Art. 4º
   (indicadores), Art. 5º (pontuação relativa, mediana mensal, FTc1), Art. 6º
   (potencializador), Art. 7º–8º (gratificação individual), Anexos I–X.
2. **Portaria CNJ nº 471/2025** — outro prêmio (Prêmio CNJ de Qualidade,
   tribunal inteiro); só serve de base para as fórmulas de Inspeções
   (Art. 9º, IX e XXV‑a), adaptadas aqui para nível de vara/mensal.
3. Painel Qlik do TJPB ("Premiação 2026") — usado para validar fórmulas com
   dados reais.

### Inconsistências do texto da Resolução (não são bugs da calculadora)

- **Inciso "VII" duplicado** no Art. 4º (Balcão Virtual e SEEU). SEEU é o card 8.
- **Medidas Socioeducativas**: Art. 4º, XI diz "15 pontos"; Anexo I diz "10
  pontos (5 Fechado + 5 Aberto)". A calculadora usa 10.
- **Art. 7º, §1º**: "50% (trinta e cinco por cento)" — algarismo e extenso
  divergem (50% vs 35%). Confirmado na imagem do PDF. Ambiguidade a esclarecer
  com Planejamento/SRH; não afeta a calculadora (que não calcula produtividade
  individual), só a orientação aos usuários.
- **Fator Potencializador**: limiar oficial é Índice de Conciliação ≥ **15%**
  (Anexo VII). Não confundir com os 18%/19% do indicador de conciliação da
  Portaria CNJ 471/2025 (outra métrica, outro prêmio).

## Como o cálculo funciona

- **Selos (Art. 3º)** sobre a pontuação relativa: Prata ≥80% · Ouro ≥90% ·
  Diamante ≥95% · Excelência ≥98%.
- **Art. 5º**: pontuação relativa = pontuação individual da unidade ÷ pontuação
  máxima aplicável. §1º: a pontuação individual é a **mediana mensal** do grau
  de cumprimento de cada indicador (Metas Nacionais ficam fora da mediana).
  A calculadora simula um único "retrato" (um mês) — principal causa de
  divergência entre o resultado dela e o do painel (a mediana de vários meses
  costuma ser menor que o mês mais recente).
- **Fórmula final** (`calc()`):
  ```
  pctBase      = Σ pontos obtidos / Σ pontos máximos aplicáveis × 100
  FTc1         = ((DA20d/Dptotal)/0,95 − 1)/100, só se negativo
  pctCorrigida = pctBase × (1 + FTc1)
  pctFinal     = pctCorrigida + (conciliação ≥15% em todos os meses ? 1 : 0)
  ```
- **FTc1** (Anexo VI): validado com dado real (75% de atendimento em 20 dias →
  −0,00211 por período, igual ao valor granular do campo `FTC_OUVIDORIA` do
  Qlik). Faixa teórica por período: −0,01 (0% no prazo) a +0,0005 (100%). O
  valor agregado exibido no painel (ex.: −0,022) é a **soma** de vários
  registros/períodos, não um cálculo único. Ajuste fino, não a principal
  alavanca.
- **Acervo Relativo** (Anexo II): `AR = (CP ÷ médiaCN₃anos) × (1 + log₁₀
  médiaCN₃anos)`. CP = processos ativos no fim do ciclo, sem suspensos/
  sobrestados/arquivo provisório. Varia ao longo do ano (CP e casos novos do
  ano corrente); os dois anos anteriores são fixos. Menor é melhor. Regra
  alternativa (grupos de 2–8 unidades, desigualdade >50%): reduzir ≥10% o
  próprio AR vs. ano anterior dá 80 pontos.
- **Tempo Médio de Pendentes Líquidos**: a Resolução só define a pontuação
  (até 115, proporcional à CB do grupo); não define a fórmula do tempo médio em
  si. A calculadora recebe o valor pronto do painel.

## Indicadores (cards) — Art. 4º / Anexo I

| Card | Nome | Máx. | Status |
|---|---|---|---|
| 1 | Acervo Relativo | 100 | ✅ validado |
| 2 | Tempo Médio Pendentes Líquidos | 115 | ✅ validado |
| 3 | IAD (baixados ÷ casos novos) | 20 | ✅ validado |
| 4 | Metas Nacionais CNJ | 60 | ✅ estrutura. Só Metas 2,5 (15 cada), 8 (10), 4,6,7,10 (5 cada). **Meta 3 não conta** neste indicador |
| 5 | % Processos Antigos | 35 | ✅ validado (denominador = Processos Ativos; ano de corte 2022) |
| 6 | Paralisados >90 dias | 50 | ✅ validado (TPA = em dia + parados) |
| 7 | Satisfação Balcão Virtual | 20 | ⚠️ não validado |
| 8 | SEEU | 100 PURA / 70 Mista | ⚠️ não validado |
| 9 | Medidas Protetivas | 10 | ⚠️ não validado (≤2 dias = cheio) |
| 10 | SNA (reavaliação) | 30 | ✅ validado (≥98%=30, 90–98%=15; sem pendência = 30) |
| 11 | Inspeções Prisionais | 15 VEP / 10 demais | meio ponto p/ atrasada; mês sem inspeção zera |
| 12 | Inspeções Socioeducativas | 10 (5+5) | ✅ aplicabilidade; unidade pode ter só Aberto |
| A | Fator Potencializador | +1 p.p. | ⚠️ não validado |
| B | Fator de Correção (FTc1) | ± | ✅ fórmula validada |

Fora do escopo: Cejusc (Anexo III), Napem (IV), Contadoria (V), Cartórios
Unificados (VIII), valores de Cláusula de Barreira (planilhas externas; o
usuário informa a do seu grupo), pesos de produtividade individual (Anexo X).

## Gratificação individual (Arts. 7º–13º) — fora da calculadora, usado em orientação

- Servidor só recebe se tiver "produtividade suficiente": ≥ X% da média das 50%
  melhores produtividades individuais da unidade (arredondando para cima o
  nº de servidores), gabinete e cartório analisados separadamente; X = 50% ou
  35% (ambiguidade acima). Chefe de cartório tem exceção (≥6 meses no cargo e
  ≥10% da média dos servidores com produtividade suficiente). Valor por selo
  (Art. 8º): Excelência 2×, Diamante 1,5×, Ouro 1×, Prata 0,5× o 1º padrão da
  classe A de Analista Judiciário. Teto de 30% do quadro do tribunal (Art. 13).
  Produtividade apurada só em PJe, SEEU e Balcão Virtual (Art. 10, Anexo X).

## Funcionalidades de UX

- Card "Pontuação Máxima Aplicável": valida a soma dos indicadores contra o
  total oficial e trava checkboxes que ultrapassariam (`updateApplicabilityLocks`).
- Card "Tipo de vara" compartilhado (`syncTipoVara`): preenche `seeu_tipo` e
  `ip_tipo` de uma vez (cada um continua editável).
- Helpers "preencher a partir da %": `fillCpFromAr`/`fillCnFromAr` (AR),
  `fillAntQtdFromPct` (% antigos), `fillFtcFromPct` (Ouvidoria; usa 100 como
  total se desconhecido — FTc1 só depende da proporção).
- Cards colapsáveis com resumo; sugestões "O que melhorar primeiro" clicáveis
  (`scrollToCard`); exportação das sugestões em PDF via `window.print()` +
  `#print_recs` (`exportarSugestoesPDF`).
- Cache: F5 mantém dados; botão "Forçar atualização completa"
  (`forcarAtualizacaoCompleta`) limpa localStorage e recarrega com `?v=timestamp`
  (cancela o `saveTimer` antes, para não "ressuscitar" dados). Ctrl+Shift+R não
  é interceptável por JS; cache de CDN do GitHub Pages está fora do controle.
- `<summary>` com fundo destacado. **Nunca** usar `display:flex` em `<summary>`
  (some o marcador e corta o texto).

## Convenções

- IDs com prefixo por indicador (`ar_`, `tp_`, `iad_`, `ant_`, `par_`, `bv_`,
  `seeu_`, `mp_`, `sna_`, `ip_`, `is_`, `conc_`, `ftc_`); cada bloco de `calc()`
  comentado `// ---- Nome ----`.
- Números sempre via `fmt(n, casas)` (pt-BR).
- Cada indicador alimenta `summaries.<chave>` (resumo do card) e `details[]`
  (sugestões).

## Pendências

- Banner descartável "Novidade" na tela inicial: oferecido, não implementado.
- Validar com dado real: SEEU, Protetivas, Balcão Virtual, Conciliação.
- Esclarecer com Planejamento/SRH: 50% vs 35% no Art. 7º, §1º.

## Deploy

Publicar = commit/push de `index.html` na branch servida pelo GitHub Pages do
repositório `gervasioteixeira/selo2026`. Sem backend, sem segredos.
