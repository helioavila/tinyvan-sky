# HANDOFF — SkyTrain Pixel (Expo Line) · Fase 1: protótipo web

> Cole este arquivo na raiz de um repositório vazio e peça ao Claude Code:
> **"Leia o HANDOFF.md, entre em plan mode e proponha o plano da Fase 0 + Fase 1 antes de codar."**

---

## 1. Visão do produto

Um visualizador em pixel art dos trens da **Expo Line** (SkyTrain, Vancouver) andando em
"tempo real". O destino final é um aparelho de mesa (LilyGO T-Display S3 AMOLED **touch**,
ESP32-S3, tela física de 240×536 usada na horizontal = 536×240, 1,91", ~307 ppi). Como a tela é
pequena e densa, a arte é desenhada numa **resolução lógica de 268×120** e cada pixel é dobrado
(2×) no aparelho. Primeiro fazemos um **protótipo web** com a mesma resolução lógica, para
acertar a estética e a lógica.

**Entrada final:** **touch capacitivo** (versão touch da placa) como entrada principal — tocar num
trem para segui-lo, tocar/deslizar para voltar. O botão físico restante serve de atalho (acordar a
tela / brilho). A tela é pequena (~44×20 mm), então as áreas de toque devem ser **generosas**:
selecionar o trem mais próximo do toque, não exigir precisão de pixel.

Dois modos de tela:

1. **Visão geral** (Fase 2): mapa da Expo Line com todos os trens se movendo.
2. **Modo seguir** (Fase 1, prioridade): o usuário escolhe um trem → o trem fica **fixo no centro da
   tela, de perfil (vista lateral / corte longitudinal)** e a cidade passa atrás dele em camadas
   de parallax.

## 2. Restrição de dados (importante)

- A TransLink **não publica a posição em tempo real do SkyTrain**. O feed GTFS-Realtime de posições
  cobre ônibus. Portanto, a posição dos trens é **calculada a partir do GTFS estático** (horários +
  traçado), interpolando entre estações pelo relógio.
- Disrupções virão depois do feed **GTFS-RT Service Alerts** (precisa de API key; Fase 3).
- Atribuições obrigatórias no rodapé/tela "Sobre":
  - "Some of the data used in this product or service is provided by permission of TransLink.
    TransLink assumes no responsibility for the accuracy or currency of the Data used in this
    product or service."
  - "© OpenStreetMap contributors" (dados de prédios/traçado, licença ODbL).
- **Não usar Google Maps / Google 3D Tiles** como fonte.

## 3. Stack

- **Vite + TypeScript + Canvas 2D**, sem framework de UI.
- Canvas lógico de **268×120**, escalado por **inteiro** (no navegador: o maior inteiro que couber
  na janela) com `imageSmoothingEnabled = false` — pixel-perfect, centralizado, fundo preto.
  Incluir um modo `?device=1` que mostra o canvas em 536×240 (2×) com uma moldura do tamanho
  físico aproximado (44×20 mm), para avaliar a legibilidade real.
- Scripts de dados em Node/TS (`scripts/`), rodando em build time. Bibliotecas sugeridas:
  um parser de CSV, `adm-zip` (ou similar) e `@turf/turf` para geometria.
- Hospedagem: **Vercel** (site estático; depois uma função serverless para os alertas).

### Regra de portabilidade (pensando no ESP32 depois)
O núcleo do renderer e da simulação (`src/core/`) **não pode depender do DOM**. Ele recebe
`tempo` + `estado` e escreve num framebuffer indexado (paleta fixa de até 32 cores, convertível
para RGB565). Camada de navegador (`src/web/`) só faz input, loop e blit para o canvas. Usar
matemática simples e dados compactos, para portar para C++/Arduino_GFX depois.

## 4. Fase 0 — pipeline de dados

**Entrada:** `data/raw/google_transit.zip` (GTFS estático da TransLink, baixado manualmente pelo
Hélio) + consultas ao Overpass API (OpenStreetMap).

**Passos (`scripts/build-data.ts`):**
1. Abrir o GTFS e **inspecionar antes de assumir**: identificar a(s) rota(s) da Expo Line
   (provavelmente por `route_long_name` contendo "Expo"; checar `route_type`). Verificar se as
   viagens usam `stop_times` completos ou `frequencies.txt` e suportar ambos.
2. A Expo Line tem **dois ramais** (King George e Production Way–University). Extrair os
   `shapes` de cada sentido/ramal.
3. Converter cada shape para metros (projeção local, ex. UTM 10N) e calcular a
   **quilometragem (chainage)** acumulada. Projetar cada estação no shape → chainage da estação.
4. Serviço do dia: resolver `calendar` + `calendar_dates` e gerar, por `service_id`, a lista
   compacta de viagens: `[[stationIdx, arrSec, depSec], …]`. Se `arr == dep`, assumir **dwell de
   ~20 s** centrado no horário.
5. **Perfil vertical** por trecho (túnel / nível do chão / trincheira / elevado / ponte): obter do
   OSM pelas vias `railway=subway|light_rail` da Expo Line com tags `tunnel`, `bridge`, `layer`,
   `cutting`. Saída: intervalos de chainage com o tipo.
6. **Prédios do corredor:** do OSM, `building` com `height` ou `building:levels` num buffer de
   ~400 m da linha. Para cada prédio: chainage (projeção no shape), lado (N/S), distância à
   linha e altura estimada (levels × 3,2 m quando faltar `height`). O **lado de visão é o Norte**
   (montanhas do North Shore ao fundo).
7. Saída em `public/data/expo.json` (traçado simplificado, estações, perfil vertical, prédios,
   viagens por serviço). Mirar em **< 1 MB**. Escrever um pequeno relatório no console
   (nº de viagens, estações, trechos, prédios).

## 5. Fase 1 — Modo seguir (entrega principal)

**Trecho do protótipo:** Waterfront → Main Street–Science World (sentido leste). É curto e mostra
três situações: **túnel** (Waterfront, Burrard, Granville), **portal** perto de
Stadium–Chinatown e **elevado** passando por BC Place / False Creek até a cúpula do Science
World. *Validar o perfil real com os dados da Fase 0 e com o Hélio.*

**Escala:** ~1 px ≈ 2 m na camada do guideway (ajustar a gosto; a tela lógica tem só 120 px de
altura — priorizar silhuetas legíveis a detalhes).

**Camadas (de trás para frente), com velocidades de parallax diferentes:**
1. Céu (gradiente em faixas, varia com a hora do dia).
2. Montanhas do North Shore (silhueta, quase paradas).
3. Prédios distantes (silhuetas mais escuras).
4. Prédios próximos (fachadas com grade de janelas gerada por código, alturas reais).
5. Guideway + pilares + postes (camada mais rápida).
6. **Trem** — fixo no centro horizontal da tela, desenhado de perfil (sprite de 2–4 carros).
7. HUD discreto no topo: nome da linha, próxima estação, sentido, ETA.

**Cenário:** montado **proceduralmente** a partir de uma biblioteca de sprites (tipos de fachada,
árvores, pilares) posicionados pelos dados de prédios por chainage. Sem bitmaps gigantes.
Marcos (BC Place, Science World, Rogers Arena) podem começar como sprites placeholder simples.

**Túnel:** fundo escuro com luzes do túnel passando em intervalos regulares; no portal, a
transição para a cidade é o "momento uau".

**Movimento:**
- Posição do trem = função do relógio e da grade de horários (interpolação entre estações).
- **Perfil de velocidade suave**: aceleração ao sair, cruzeiro, frenagem ao chegar (ex. curva
  trapezoidal ou smoothstep sobre o tempo entre paradas) — nada de velocidade linear.

**Estações:**
- Ao chegar, a estação (cobertura, pilares, placa com o nome em pixel art) entra como **camada
  frontal que oculta a cidade**. O trem fica **parado e estático** durante o dwell (sem animar
  portas). Na saída, a estação desliza para fora.
- Enquanto a estação cobre a tela, trocar o "bairro" do cenário de fundo, se necessário.

**Hora do dia:** à noite, céu escuro e janelas acesas (o AMOLED fica lindo com fundo preto).

**Controles de desenvolvimento (web):**
- Relógio simulado: `?t=08:15` para fixar o horário, `?speed=10` para acelerar.
- Teclas: `←/→` troca de trem, `Espaço` pausa, `D` overlay de debug (chainage, velocidade, FPS).
- Clique/toque na tela: sair do modo seguir (preparando a interação touch da Fase 2).

**Critérios de aceite da Fase 1:**
- [ ] Roda a 60 fps estáveis no desktop, pixel-perfect, em 268×120 escalado por inteiro.
- [ ] Modo `?device=1` mostra como fica no tamanho físico da tela.
- [ ] O trem segue os horários reais do GTFS do dia (conferível no overlay de debug).
- [ ] Túnel → portal → elevado com parallax visível e convincente.
- [ ] Parada em cada estação do trecho, com oclusão e trem estático.
- [ ] Núcleo em `src/core/` sem nenhuma referência ao DOM.
- [ ] Deploy funcionando na Vercel.

## 6. Próximas fases (não fazer agora; só não bloquear)

- **Fase 2 — Visão geral + transição:** mapa da Expo Line (estilo a definir: plano minimalista ou
  volumetria isométrica com prédios em blocos de 3 tons). Tocar num trem (o mais próximo do
  toque) → zoom → a vista gira para o perfil → modo seguir; tocar/deslizar → volta. No web,
  clique do mouse simula o toque.
- **Upgrade futuro — 3D estilo PS1:** trocar o renderer 2D por cena low-poly (prédios extrudados
  do OSM, guideway, trem low-poly, neblina, texturas affine, vertex snapping, dithering). Web em
  Three.js; aparelho com o rasterizador por software **Jet** (C++, ESP32-S3). Por isso: **manter
  dados, horários e simulação de posição (chainage + tempo) totalmente separados do renderer**,
  para que o renderer 2D possa ser substituído sem mexer no resto.
- **Fase 3 — Alertas:** função serverless `/api/alerts` na Vercel que consulta
  `https://gtfsapi.translink.ca/v3/gtfsalerts?apikey=…` (chave em variável de ambiente
  `TRANSLINK_API_KEY`, **nunca no cliente**), filtra a Expo Line e devolve JSON enxuto, com cache
  de ~60 s.
- **Fase 4 — Atualização de dados:** GitHub Action semanal que roda `build-data` e faz commit do
  JSON → a Vercel faz redeploy sozinha.
- **Fase 5 — Porte para o ESP32-S3:** Arduino/PlatformIO + **Arduino_GFX** (a TFT_eSPI não
  serve para o painel QSPI RM67162). Modo "espiar" (tela acorda por toque), Wi-Fi econômico,
  desligar de madrugada, deslocamento periódico de 1 px contra burn-in.

## 7. Como trabalhar

- Começar por **plan mode**; mostrar o plano e a estrutura de pastas antes de codar.
- Na Fase 0, **imprimir o que o GTFS realmente contém** antes de decidir o formato final.
- Commits pequenos por etapa. README com como rodar (`npm run data`, `npm run dev`) e as
  atribuições.
- Se faltar dado (ex. altura de prédios), usar um fallback plausível e registrar no relatório,
  sem travar.
