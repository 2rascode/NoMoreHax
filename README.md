# NoMoreHax — Anticheat para Minecraft 1.8.9

Anticheat **server-side** baseado em **análise de pacotes** (ProtocolLib) para servidores
Spigot/Paper **1.8.8** (protocolo 47, clients 1.8.x incluindo 1.8.9). Detecta trapaças
pelo tráfego do jogador, então funciona contra qualquer client — não é um mod e não
depende de nada instalado no lado do jogador.

## Como funciona

Todo pacote de entrada relevante passa por um único `PacketListener` (ProtocolLib), que
mantém um *snapshot* de movimento/rotação por jogador e o entrega aos checks. Cada check
acumula **violation level (VL)**; ao cruzar limiares configuráveis, o `PunishmentManager`
executa comandos (log → kick → ban). VL **decai** com jogo limpo, e há **isenções**
automáticas para estados legítimos (entrar, teleporte, respawn, troca de mundo, knockback)
e **relaxamento sob lag** (TPS baixo).

```
Pacote → PacketListener → PlayerData (snapshot) → Checks → VL → Alerta / Punição
```

## Checks incluídos

| Categoria  | Check        | Sinal principal |
|------------|--------------|-----------------|
| Combate    | Reach        | Distância olho→hitbox além do alcance vanilla; **cancela o hit impossível** (compensação por transaction-ping) |
| Combate    | KillAura     | Ângulo do golpe fora do campo de visão + *multi-aura* (2+ alvos no mesmo tick) |
| Combate    | AutoClicker  | CPS impossível + baixo desvio-padrão dos intervalos (clique robótico) |
| Combate    | Velocity     | Knockback recebido menor que o aplicado (janela de 3 ticks) |
| Combate    | Aim          | Análise de GCD/sensibilidade da rotação (aimbot) — *experimental, off por padrão* |
| Movimento  | Speed        | **Predição por fricção vanilla** (velocidade anterior × fricção + input) |
| Movimento  | Fly          | **Predição por gravidade vanilla** (motionY: −0.08 accel, ×0.98 drag) — pega hover, glide e ascensão |
| Movimento  | NoFall       | Flag `onGround` falsificada estando no ar caindo |
| Movimento  | Timer        | **Relógio por transaction** (à prova de jitter), não relógio de parede |
| Movimento  | Sprint       | Estados de corrida impossíveis (sprint + sneak / fome≤6 / usando item) |
| Movimento  | NoSlow       | Velocidade cheia enquanto usa item que deveria desacelerar (comer/beber/arco/bloquear) |
| Mundo      | Nuker        | Blocos quebrados por segundo / múltiplos no mesmo tick |
| Mundo      | FastBreak    | Bloco com dureza quebrado rápido demais |
| Mundo      | Scaffold     | Colocar blocos abaixo dos pés, em movimento, em alta cadência (bridging) |
| Pacote     | BadPackets   | Coordenadas NaN/fora do mundo, pitch ilegal, agir estando morto (VL alto) |
| Pacote     | PacketOrder  | Uso-duplo de item no mesmo tick + ação antes do pacote de client settings |
| Pacote     | Post         | Ação enviada **depois** do movimento no tick (blink/reach/aura) — via transaction |
| Utilitário | AntiBot      | Filtro de flood de entrada exigindo o pacote de client settings |

## Melhorias inspiradas no GrimAC (v1.1)

Após comparar com o GrimAC, portamos as ideias de maior impacto sem reescrever a
engine de predição completa dele:

- **Compensação de latência por transaction:** o servidor envia um pacote de
  transaction por tick e mede o *round-trip* real. Esse ping (imune a jitter)
  alimenta o Timer e a tolerância do Reach — muito mais confiável que o ping do
  cliente. (`TransactionManager`)
- **Timer amarrado ao relógio do jogador:** um cliente que buferiza pacotes no lag
  e depois dispara em rajada não consegue mais "poupar crédito" para burlar —
  o balanço só pode ficar atrás do tempo real pelo tanto do ping medido.
- **Setback (rubber-band):** violações de movimento puxam o jogador de volta à
  última posição válida, anulando o efeito do cheat na hora — não é só VL.
  (`SetbackManager`)
- **Cancelamento de pacote:** hits de Reach fisicamente impossíveis e pacotes
  inválidos (posição NaN/fora do mundo, pitch ilegal) são **descartados**, então
  a ação ilegal nunca registra.
- **Checks de movimento preditivos:** Speed e Fly usam o modelo físico do vanilla
  1.8 em vez de tetos fixos — mais precisos e com menos falso positivo.

## Sub-checks estilo Grim (v1.2)

Novos checks que ampliam a cobertura, lendo o estado do jogador a partir de
pacotes de ação (`ENTITY_ACTION`, uso de item) e do snapshot da thread principal:

- **Sprint** — corrida em estados que o vanilla proíbe (com sneak, fome ≤ 6, ou
  usando item). Determinístico, com buffer de 3 ticks contra desync.
- **NoSlow** — velocidade no chão acima do teto de "usando item" durante uma
  janela de uso confirmada (place de uso-no-ar → release).
- **Aim** — GCD dos deltas de pitch: rotação humana quantiza pela sensibilidade,
  aimbot não. Experimental (`enabled: false`), precisa de tuning por servidor.
- **PacketOrder** — uso-duplo de item no mesmo tick e ações antes do client
  settings (bot). Determinístico.

## Framework de tick-por-transaction + Post (v1.3)

O salto de arquitetura do Grim, agora portado. A cada pacote de movimento (fim do
tick do cliente) o servidor envia **inline, na thread de rede**, uma *transaction*
(window-confirm, id negativo para não colidir com o inventário). Como os pacotes
são estritamente ordenados no TCP, a resposta do cliente chega logo depois do
movimento e antes das ações do próximo tick — um marcador de fim-de-tick confiável
e compensado por latência. Isso alimenta:

- **Ping real** (round-trip) → tolerância do Reach e piso do Timer.
- **Contador de tick lag-compensado** (`transactionTick`).
- **Post** (`checks/packet/Post.java`): máquina de estados fiel ao Grim —
  movimento marca `sentFlying`, ações seguintes entram na fila, e a resposta da
  transaction fecha o tick; se sobrou ação na fila, ela foi enviada *depois* do
  movimento (blink, reach por reordenação, alguns auras). O `clear()` a cada
  movimento é auto-corretivo contra lag; ainda exige TPS saudável, não-isenção e
  um buffer de 4 posts seguidos, então jitter de rede não gera falso positivo.

O `ARM_ANIMATION` (swing) fica de fora do Post de propósito — o ViaVersion atrasa a
animação de swing de clients 1.8 e falsificaria. Continua de fora (precisaria da
engine de predição completa): as variantes de PacketOrder que dependem de rewind.

## O que foi "melhorado" além do núcleo

- **Compensação de latência**: Reach ganha tolerância proporcional ao ping.
- **Decaimento de VL**: jogo limpo remove VL ao longo do tempo (config).
- **Isenções de estado**: join/teleporte/respawn/mundo/knockback não geram falso positivo.
- **Relaxamento sob lag**: checks de movimento pausam abaixo de um TPS mínimo.
- **Buffers de violação**: checks de movimento exigem infração sustentada, não um único tick.
- **Análise estatística de cliques**: desvio-padrão além de só CPS.
- **Punições idempotentes**: cada tier dispara uma vez por sessão (não re-bane no decaimento).
- **Config por-check** com `/nomorehax reload` a quente, canais de **alerts** e **verbose**.
- **Robustez**: erros de parsing de pacote nunca derrubam o pipeline de rede.

## Build

Requer **JDK 8+**, **Maven** e internet (baixa `spigot-api` 1.8.8 e `ProtocolLib`
4.5.1 via JitPack — a última linha do ProtocolLib compatível com 1.8). Na raiz:

```bash
mvn clean package
```

O jar sai em `target/NoMoreHax-1.0.0.jar`.

> Verificação: todos os 38 arquivos-fonte compilam sem erros contra
> `spigot-api 1.8.8-R0.1` + `ProtocolLib 4.5.1` (a versão exata de runtime), e a
> lógica estatística do núcleo (`MathUtil`, usada por AutoClicker/KillAura) passa
> em testes unitários.

## Instalação

1. Instale o **ProtocolLib** (compatível com 1.8.8) na pasta `plugins/`.
2. Copie `NoMoreHax-1.0.0.jar` para `plugins/`.
3. Inicie o servidor. Edite `plugins/NoMoreHax/config.yml` e rode `/nomorehax reload`.

## Comandos e permissões

- `/nomorehax alerts` — liga/desliga alertas (`nomorehax.alerts`)
- `/nomorehax verbose` — liga/desliga verbose de debug (`nomorehax.verbose`)
- `/nomorehax vl <jogador>` — mostra os VLs atuais
- `/nomorehax info` — status do plugin
- `/nomorehax reload` — recarrega a config (`nomorehax.command`)
- `nomorehax.bypass` — isenta o jogador de todos os checks (padrão: ninguém)

## Notas honestas de precisão

Checks **heurísticos** (Speed, Fly, NoFall, Scaffold) usam limiares + buffers e podem
precisar de ajuste fino por servidor — o Speed já considera `walkSpeed`, poção de Speed e
gelo. Checks **determinísticos** (BadPackets, multi-aura, Timer, Reach) são de altíssima
confiança. Comece com punições em modo *log*, observe o `verbose`, e só então ative kick/ban.

**Nenhum anticheat pega 100% nem é infalível** — quem promete isso está errado. O que este
projeto garante é robustez de engenharia: o estado do mundo (near-ground, meio, gelo, poção,
gamemode) é fotografado uma vez por tick na **thread principal** (`SnapshotTask`) e os checks
assíncronos só leem campos `volatile` — nenhum carregamento de chunk ou iteração de entidades
acontece na thread de rede, então um pacote malformado não trava o servidor. Estruturas
compartilhadas entre threads são concorrentes, e erros de parsing são contidos por `try/catch`.
