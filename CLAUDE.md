# CLAUDE.md

Guia para o Claude Code (claude.ai/code) trabalhar neste repositório.

## Projeto

"Survivor Browser" é um jogo de navegador *roguelite* com visão de cima (estilo Vampire Survivors e Megabonk). O jogador sobrevive a hordas, coleta XP, sobe de nível e escolhe armas e passivas. Ao morrer, recomeça, mas as **moedas douradas** ficam para a Loja.

JavaScript puro com Canvas 2D, **sem build, sem dependências e sem package.json**. Publicado como arquivos estáticos no GitHub Pages (https://eduardo-macedo07.github.io/Prontuario/).

Todo texto de interface, comentário de código, README e mensagem de commit é em **português (pt-BR)**. O `README.md` descreve todas as regras do jogo em detalhe (fases, itens, loja, desafios, personagens): consulte-o antes de mudar o balanceamento.

> O nome do repositório antigo era `Prontuario`. O conteúdo é este jogo, não um sistema de prontuário.

## Rodando e testando

- **Rodar:** abra `index.html` direto no navegador (os scripts são `<script>` clássicos, não módulos, então `file://` funciona). Ou sirva a pasta com `python3 -m http.server`.
- **Sem ferramentas:** não há linter, testes automáticos nem build.
- **Checagem rápida de sintaxe:**
  ```
  node --check js/game.js && node --check js/art.js && node --check js/audio.js && node --check assets/sprites.js
  ```
- **Teste no navegador:** use Playwright headless (disponível globalmente). Rode com `NODE_PATH=$(npm root -g) node script.js` e mantenha scripts e capturas de tela **fora do repositório**.
- **Ganchos de teste:** `game.js` expõe `window.__jogo` com:
  - leitura: `state`, `player`, `enemies`, `structures`, `weapons`, `save`, `lanes`, `run`, `time`, `zoom`, `lowQuality`, `eshots`, `strikes`, `summons`, `pickups`;
  - ações: `startGame`, `selectChar`, `selectStage`, `addWeapon`, `addItem`, `gainXp`, `spawnEnemy`, `spawnElite`, `spawnBoss`, `killEnemy`, `setTime(v)`, `buildChoices`, `buildItemChoices`, `openNearChest`, `rollRarity`, `completeChallenge`, `checkChallenges`, `persist`.
  - Diferente de outros projetos, **não há `step(dt)`**: o jogo roda por `requestAnimationFrame`. Para acelerar um teste, use `setTime` e dê armas e nível com `addWeapon` e `gainXp`.
- **Viewports:** teste um tamanho de desktop (ex.: 1366x768) e um de celular em paisagem (ex.: 844x390). O joystick virtual e o zoom da câmera só aparecem em telas de toque (`pointer: coarse`).

## Arquitetura

Ordem de carga em `index.html` (scripts compartilham globais): `assets/sprites.js` → `js/audio.js` → `js/art.js` → `js/game.js`.

### `js/game.js` (cerca de 4.800 linhas, uma IIFE)

Contém dados, simulação, UI e o loop principal. Seções principais, na ordem do arquivo:

- **Canvas e câmera:** `resize()` calcula `W`, `H` e `zoom` (em telas pequenas a câmera se afasta, mínimo 0,6).
- **Save:** `localStorage`, chave `survivorbrowser.save`. `defaultSave()` define o formato (`coins`, `chars`, `bought`, `stages`, `challenges`, `unlocks`, `stats`, `settings`...); `loadSave()` faz `merge` com o salvo e migra versões antigas; `persist()` grava. Ao mudar o formato, adicione o campo em `defaultSave()` (o `merge` preenche o que falta em saves antigos).
- **Dados (tabelas no topo):** `CHARACTERS`, `SHOP`, `PERMS` (melhorias permanentes), `CHALLENGES`, `WEAPONS`, `PASSIVES`, `BOONS` (bênçãos das fogueiras), `ITEMS` (itens de baú), `RARITY`/`TIERS`, `ENEMY_TYPES`, `STAGES`, `DIFFS`. É aqui que se faz o balanceamento.
- **Estado da partida:** variáveis soltas (`player`, `enemies`, `projectiles`, `pickups`, `weapons`, `run`...) zeradas em `resetWorld()`. `run` guarda os dados da partida usados por desafios, adaptação e Ameaça.
- **UI:** HUD e menus são overlays DOM definidos em `index.html`. `setState(s)` troca entre `OVERLAYS` (`home`, `stages`, `chars`, `shop`, `challenges`...). Os estados de jogo são `home`, `playing`, pausado, escolhendo recompensa e `dead`.
- **Loop:** `frame(now)` limita `dt` a 0,05 s e chama `update(dt)` e `updateEffects(dt)` só em `state === 'playing'`. `watchPerformance` ativa o modo de baixa qualidade (`lowQuality`, menos partículas) se o FPS ficar abaixo de 40 por 3 s.
- **Ordem de `update(dt)`:** movimento do jogador (teclado ou joystick) → regeneração → `updateSpawning` → `updateStructures` → `updateSummons` → `updateWeapons` → `updateProjectiles` → `updateRings` → `updateHazards` → `updateEnemies` → `updateStrikes` → `updateLanes` → `updateMeteors` → `updateEnemyHazards` → `updateSpins` → limpa inimigos mortos → `updatePickups` → itens de relógio → `checkChallenges` (a cada 0,5 s) → `openChoices` se houver níveis, baús ou santuários pendentes.

### Regras que atravessam várias partes

- **Armas:** cada arma tem `max` (nível 10), `base` e `ups` (lista de bônus por nível). O comportamento fica em `WEAPON_UPDATE[id](w, dt)`, e o alcance em `RANGE`. A mira usa `aimTarget`/`aimTargets`; arma só dispara com inimigo no alcance.
- **Raridade:** a carta multiplica o bônus do nível (Comum 1×, Raro 1,67×, Épico 2,67×, Lendário 3,83×). Cartas de arma raras ganham bônus extras (`weaponExtras`). `rarityWeights`/`rollRarity` dependem de Sorte.
- **Dano e morte:** todo dano a inimigo passa por `damageEnemy`, e a morte por `killEnemy` (XP, moedas, chaves, drops, contagem de desafios). Efeitos de item (`itemProcs`) têm trava `inProc` para não disparar em cadeia.
- **Inimigos:** `ENEMY_TYPES` guarda os atributos e `ENEMY_BEHAVIOR` guarda o comportamento especial de cada tipo (bote, investida, tiros, explosão). `spawnEnemy`, `spawnElite`, `spawnBoss` e `spawnFinal` criam os tipos; `STAGES[id].pick(t, r)` escolhe o tipo comum conforme o tempo.
- **Fases e dificuldades:** `STAGES` define bioma, chefões (`bosses`), chefão final (`final`) e `pick`. `DIFFS` multiplica vida, dano, spawn e moedas. Vencer o chefão final chama `stageClear`, que abre o portal; `enterPortal` troca de dificuldade mantendo a build. Depois do chefão final, a **Ameaça** (`threat()`) sobe a cada 30 s.
- **Adaptação à build:** `adaptFeedback(e)` compara o tempo que um mini-chefe, chefão ou chefão final levou para morrer com `TTK_TARGET` e ajusta `run.adapt` (até `ADAPT_MAX`, nunca abaixo de 1). `adaptHp(kind)` aplica o fator.
- **Recompensas:** `buildChoices(isChest)` monta as cartas de nível, `buildItemChoices` as de baú, `buildBoons` as de fogueira. A Loja é montada por `shopItems`/`renderShop`, com `priceOf`, `isOwned` e requisitos (`reqMet`).
- **Desafios:** lista `CHALLENGES`, avaliados por `checkChallenges()`; `completeChallenge(id)` paga as moedas.

### `js/art.js` (cerca de 3.900 linhas, IIFE `Art`)

Todo o desenho é procedural, e `Art` exporta funções como `drawHero`, `drawEnemy`, `drawGround`, `propsInView`, `drawProp`, `portal`, `drawChest`, `setBiome`, `stagePreview`.

- **Sprites em cache:** `sprite(key, size, draw)` guarda o desenho em um canvas; `ENEMY_DRAW[type]` e `HEROES[id]` são as funções de desenho de cada inimigo e personagem.
- **Cenário infinito:** o mapa é gerado por chunks de 512 unidades (`CHUNK`, `chunkPaths`, `hash01` e `rng` com semente), com `BIOMES` e `PROPS` por bioma. `setBiome(id, variante)` troca o visual (a variante muda a cara de cada dificuldade).
- **Sprites do usuário:** `assets/sprites.js` define `window.SPRITES`. Uma entrada ativa (sem `//`) troca o desenho por código por um PNG de `assets/sprites/` (`SHEETS`/`drawSheet`). Detalhes de formato em `assets/LEIA-ME.md`.
- **Ferramentas de sprites:** `ferramentas/*.py` geram o PDF modelo (`gerar_pdf_modelo.py`) e convertem um PDF preenchido em sprites (`pdf_para_sprites.py`).
- **Ícones:** `Art.icon`, `heroIcon` e `stagePreview` geram imagens para a loja e os menus. Aqui **há** uso de `toDataURL` (diferente de outros projetos).

### `js/audio.js` (IIFE `Sound`)

SFX e músicas procedurais com Web Audio, sem arquivos de áudio. API: `Sound.unlock()` (chamar depois de um gesto do usuário), `Sound.play(nome)`, `Sound.music(...)`, `setVolumes`, `duck`. Nomes de SFX em uso: altar, arrow, bonfire, boss, buy, chest, click, coin, death, denied, explosion, fire, gem, growl, heal, hit, hurt, ice, knife, levelup, select, swarm, tremor, warn, whistle, zap. Cada fase e dificuldade tem música própria (`updateMusic` em `game.js`).

## Como adicionar coisas

- **Nova arma:** entrada em `WEAPONS` (`base`, `ups`), alcance em `RANGE`, comportamento em `WEAPON_UPDATE`, e, se for da Loja, item em `SHOP` e `isUnlocked`. Se criar projétil novo, trate-o em `updateProjectiles`.
- **Novo inimigo:** tipo em `ENEMY_TYPES`, comportamento em `ENEMY_BEHAVIOR`, desenho em `ENEMY_DRAW` (`art.js`), e inclua-o no `pick` da fase.
- **Nova fase:** entrada em `STAGES`, bioma em `BIOMES`/`PROPS` (`art.js`), tier em `STAGE_TIER`, música em `updateMusic`, e desafios ligados a ela.
- **Novo personagem:** entrada em `CHARACTERS`, função em `HEROES` (`art.js`), item em `SHOP`, campo em `defaultSave().chars` e, se houver passiva própria, a lógica em `game.js`.
- **Novo item de baú:** entrada em `ITEMS`; se for com chance de disparo, use `procRoll` em `itemProcs`.

## Convenções

- **Cache-busting:** depois de mudar JS ou CSS, suba o `?v=` das tags de `index.html` (hoje `202610140232`) para o mesmo número em todas.
- **Texto e código em pt-BR**, com comentários curtos explicando o porquê.
- **Estilo visual:** fontes Cinzel (títulos) e Nunito (texto), tema escuro roxo. O texto do canvas usa as mesmas fontes (`TITLE_FONT`, `FONT`).
- **Desempenho:** celulares são alvo. Evite alocar objetos por quadro em laços quentes e respeite `lowQuality` e `maxParticles` ao criar partículas.
- **Balanceamento:** depois de mudar números, jogue uma partida em cada fase e dificuldade afetada. O jogo já ajusta a dificuldade dos chefões pela adaptação à build, então revise `TTK_TARGET` e `ADAPT_MAX` junto.
- **Não commitar** capturas de tela nem scripts de teste.
