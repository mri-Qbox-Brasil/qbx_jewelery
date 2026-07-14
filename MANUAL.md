# qbx_jewelery — Manual

Roubo à joalheria Vangelico: hackeia a caixa elétrica para destrancar a porta, quebra as vitrines com arma na mão e leva as joias, disparando alarme e alerta policial.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Fluxo do roubo](#fluxo-do-roubo)
5. [Integrações](#integrações)
6. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
7. [Localização](#localização)
8. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | `GetPlayer`, `Notify`, `GetDutyCountType('leo')`, `GetQBPlayers` |
| `ox_lib` | Sim | Callbacks, zonas, textUI, locale |
| `ox_inventory` | Sim | Checagem/consumo do item de hack e entrega das joias |
| `ox_doorlock` | Sim | Porta da joalheria é destrancada e retrancada por `getDoorFromName` / `ox_doorlock:setState` |
| `ultra-voltlab` | Sim | Minigame do hack da caixa elétrica. Sem ele, o hack nunca conclui |
| `ox_target` | Não | Só quando `useTarget = true` |
| `qbx_policejob` | Não | Recebe `police:server:policeAlert` e `evidence:server:CreateFingerDrop` |
| `qb-scoreboard` | Não | Recebe `qb-scoreboard:server:SetActivityBusy` para marcar a joalheria como ocupada |

---

## Instalação

1. Copie a pasta `qbx_jewelery` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_jewelery
   ```
3. Cadastre no `ox_doorlock` a porta da joalheria com o nome definido em `doorlock.name` (padrão `vangelico_jewellery`). Sem essa porta cadastrada, o hack quebra ao tentar destrancar.
4. Garanta que os itens existam no `ox_inventory`:
   - Item do hack: `electronickit`
   - Recompensas: `rolex`, `diamond_ring`, `goldchain`, `10kgoldchain`
5. **Conflitos** — não rode junto com o `qb-jewelery` original: os dois usam os mesmos eventos e callbacks (`qb-jewelery:*`).

---

## Configuração

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `fingerprintChance` | number | Sim | Chance (0–100) de deixar digital ao quebrar uma vitrine. Não deixa digital se estiver de luvas |
| `useDrawText` | bool | Sim | `true` usa texto 3D no mundo; `false` usa `lib.showTextUI` |
| `useTarget` | bool | Sim | `true` usa `ox_target` na caixa elétrica e nas vitrines; `false` usa proximidade + tecla `E` |
| `alarmDuration` | ms | Sim | Duração do alarme `JEWEL_STORE_HEIST_ALARMS` (padrão 240000 = 4 min) |

### `config/server.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `timeOut` | ms | Sim | Cooldown após o alarme. Ao expirar, a porta é retrancada e todas as vitrines voltam ao estado íntegro (padrão 2700000 = 45 min) |
| `minimumPolice` | number | Sim | Policiais em serviço (`leo`) necessários para iniciar o hack |
| `notEnoughPoliceNotify` | bool | Sim | Notifica o jogador quando não há policiais suficientes |
| `reward.minAmount` | number | Sim | Mínimo de sorteios de item por vitrine quebrada |
| `reward.maxAmount` | number | Sim | Máximo de sorteios de item por vitrine quebrada |
| `reward.items` | array de `{name, min, max}` | Sim | Pool de joias. A cada sorteio, um item é escolhido e entregue em quantidade `math.random(min, max)` |
| `allowedWeapons` | tabela `[hash] = true` | Sim | Armas que permitem quebrar a vitrine. O padrão cobre SMGs, escopetas e rifles |

### `config/shared.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `electrical` | `vec4` | Sim | Posição e heading da caixa elétrica (prop `tr_prop_tr_elecbox_01a`, criado pelo cliente) |
| `doorlock.name` | string | Sim | Nome da porta no `ox_doorlock` |
| `doorlock.hackTime.min` / `.max` | number | Sim | Faixa de tempo (segundos) sorteada para o minigame do `ultra-voltlab` |
| `doorlock.requiredItem` | string | Sim | Item exigido para hackear (padrão `electronickit`) |
| `doorlock.loseItemOnUse` | bool | Sim | Consome o item ao iniciar o hack |
| `vitrines` | array | Sim | Vitrines da loja. Ver abaixo |

#### Formato de uma vitrine

```lua
{coords = vec3(-626.83, -235.35, 38.05), isOpened = false, isBusy = false, rayFire = 'DES_Jewel_Cab3', heading = 36.17}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `coords` | `vec3` | Posição da vitrine |
| `heading` | number | Heading que o personagem assume ao quebrar |
| `rayFire` | string | Nome do objeto RayFire quebrável do mapa. `DES_Jewel_Cab4` usa a animação frontal; os demais usam a animação de cima |
| `isOpened` / `isBusy` | bool | Estado inicial — mantenha `false` |

O config traz um conjunto alternativo de vitrines (K4AMBI) comentado no final do arquivo, para quem usa a MLO da K4MB1.

---

## Fluxo do roubo

1. O jogador chega à caixa elétrica com um `electronickit`. O servidor valida o item, a distância (2m) e a quantidade de policiais em serviço.
2. O item é consumido (se `loseItemOnUse`) e o minigame do `ultra-voltlab` roda por um tempo sorteado entre `hackTime.min` e `hackTime.max`, com cena sincronizada de animação na caixa.
3. Sucesso destranca a porta via `ox_doorlock`. Falha ou timeout apenas libera a caixa para uma nova tentativa.
4. Dentro da loja, com uma arma permitida em mãos, o jogador quebra as vitrines. Cada quebra roda a animação, o efeito RayFire, a partícula e o som — sincronizados para todos os jogadores num raio de 20 metros.
5. Cada vitrine quebrada entrega joias aleatórias e pode deixar uma digital. A primeira vitrine quebrada dispara o alarme e o alerta policial.
6. Após `timeOut` ms do alarme, a porta é retrancada, as vitrines são restauradas e a joalheria fica disponível novamente.

O raio de 80 metros em torno da primeira vitrine controla a criação da caixa elétrica e a manutenção do estado visual das vitrines já quebradas.

---

## Integrações

### ox_doorlock

A porta com o nome de `doorlock.name` é destrancada (`state 0`) no hack bem-sucedido e retrancada (`state 1`) quando o cooldown do alarme expira.

### ultra-voltlab

O minigame do hack é chamado via `TriggerEvent('ultra-voltlab', tempo, callback)`. O callback recebe `result`: `0` falha, `1` sucesso, `2` tempo esgotado, `-1` erro.

### qbx_policejob

Alerta enviado em `police:server:policeAlert` na primeira vitrine quebrada. Digitais deixadas via `evidence:server:CreateFingerDrop`.

### qb-scoreboard

A joalheria é marcada como ocupada (`SetActivityBusy 'jewellery', true`) durante o alarme e liberada quando o cooldown expira.

---

## Entrypoints para outros recursos

### Callbacks (`lib.callback`) — servidor

```lua
-- Valida item, polícia e distância; consome o item e libera o hack
lib.callback('qb-jewelery:callback:electricalbox', false, cb)

-- Valida arma, estado da vitrine e distância; libera a quebra
lib.callback('qb-jewelery:callback:cabinet', false, cb, vitrineIndex)
```

### Eventos de servidor

```lua
TriggerServerEvent('qb-jewelery:server:endcabinet')        -- conclui a quebra, entrega joias, dispara alarme
TriggerServerEvent('qb-jewellery:server:succeshackdoor')   -- hack bem-sucedido: destranca a porta
TriggerServerEvent('qb-jewellery:server:failedhackdoor')   -- hack falhou: libera a caixa elétrica
```

### Eventos de cliente

```lua
TriggerClientEvent('qb-jewelery:client:alarm', -1)                  -- inicia o alarme
TriggerClientEvent('qb-jewelery:client:syncconfig', -1, vitrines)   -- sincroniza o estado das vitrines
```

---

## Localização

Strings via `ox_lib` locale, em `locales/`:

`cs`, `en`, `fr`, `pt-br`

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
qbx_jewelery/
├── client/
│   └── main.lua          — caixa elétrica, minigame do hack, quebra das vitrines (RayFire, partícula, som), alarme
├── server/
│   └── main.lua          — validações, consumo do item, doorlock, recompensas, alarme e cooldown
├── config/
│   ├── client.lua        — digital, target/drawtext, duração do alarme
│   ├── server.lua        — cooldown, polícia mínima, pool de joias, armas permitidas
│   └── shared.lua        — caixa elétrica, doorlock, lista das 20 vitrines
├── locales/              — cs, en, fr, pt-br (.json)
└── fxmanifest.lua
```
