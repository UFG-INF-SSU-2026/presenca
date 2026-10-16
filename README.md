# Presença

**Registro automático e proporcional de frequência em sala de aula.**

O sistema detecta a permanência física do estudante na sala durante a janela da aula e
**devolve ao professor a lista de presença já preenchida**, sem que ninguém execute a
chamada. Ao estudante, avisa em tempo hábil quando uma ausência em curso está prestes a
lhe custar presença.

---

Projeto da disciplina **Software para Sistemas Ubíquos**
Instituto de Informática — Universidade Federal de Goiás
Prof. Dr. Otávio Calaça Xavier

## Integrantes

| Nome | Matrícula |
|---|---|
| Alany Gabrielly | 202105018 |

## A ideia em uma frase

A chamada consome de 3 a 5 horas-aula por semestre, mede um instante arbitrário em vez
da aula inteira e é trivialmente fraudável. Este sistema troca **presença** (binária,
medida num instante) por **permanência** (contínua, acumulada ao longo da aula).

## Como funciona

1. **Três âncoras ESP32** ficam fixas na sala, transmitindo um sinal identificado e um
   token que se renova a cada 10–15 segundos. Elas formam uma malha ESP-NOW entre si e
   elegem uma líder, que é a única a falar com o gateway.
2. **O aplicativo do estudante** ouve as três âncoras, mede a intensidade de sinal de
   cada uma e envia ao servidor um pacote assinado. As âncoras nunca sabem quem está na
   sala — a identidade trafega apenas entre aparelho e servidor, cifrada.
3. **O servidor** trilatera a posição, testa se ela está dentro do polígono da sala e
   alimenta uma máquina de estados que acumula o tempo de permanência.
4. **Quando uma ausência passa da metade do limite**, o sistema notifica o estudante —
   *"você está fora há 10 minutos, seu limite é 15"*. É a notificação que fecha o laço:
   o estudante retorna e altera a própria posição, que é a grandeza medida.
5. **Ao fim da aula**, o professor recebe a lista preenchida e uma fila curta de
   exceções para revisar.

## Decisões de projeto

| Decisão | Motivo |
|---|---|
| **Permanência, não presença** | O erro do sistema passa a ser proporcional à falha: três minutos de sensor ruim custam três minutos, não a aula inteira |
| **Âncora só transmite, nunca escuta** | A infraestrutura da sala não guarda nenhum dado pessoal. Arrancada da parede, não revela nada |
| **Trilateração com três âncoras** | Rádio não respeita parede. Um raio é um círculo; uma sala é um polígono. Três âncoras permitem testar pertinência ao polígono, não distância a um ponto |
| **QR lido uma única vez** | Pareia pessoa e chave criptográfica no início do semestre. Depois disso, cerca de vinte aulas sem nenhuma ação do estudante |
| **Identidade por chave, não por hardware** | MAC é randomizado por rede, IMEI é inacessível, ID de anúncio é zerável. A chave nasce no cofre de hardware do aparelho e só muda quando o aparelho muda |
| **Orçamento de ausência por sessão** | 15 minutos de tolerância no total da aula, não por evento — senão basta sair repetidamente por pouco tempo |
| **Avisar antes de punir** | Notificação após a falta consolidada é informação. Antes, é atuação |
| **Malha ESP-NOW entre âncoras** | Reduz a carga no ponto de acesso a um terço e permite reorganização automática quando um nó falha |

## Classificação

| Categoria | Situação |
|---|---|
| **Aplicação ubíqua** | ✅ principal — interação implícita, ciência de contexto, infraestrutura invisível |
| **Internet das Coisas** | ✅ dispositivos conectados, MQTT, processamento remoto |
| **Rede de sensores** | ✅ medição cooperativa, localização por âncoras, nó móvel, multi-salto |
| **Sistema ciber-físico** | ✅ com humano no laço — a notificação atua sobre a posição medida |

Justificativas completas em [`docs/atividade-01.md`](docs/atividade-01.md), item 8.

## Risco principal

**Privacidade.** Não pelo volume de dado, que é pequeno, mas por um subproduto que o
sistema obtém de graça sem precisar: o **grafo de colocalização** — quem senta perto de
quem ao longo do semestre. Ele nunca é materializado nem persistido.

O compromisso de fundo, registrado no documento: *neste domínio, ubiquidade e privacidade
são grandezas inversas*. As versões mais invisíveis do sistema — câmera com
reconhecimento facial, sensor de pressão no piso — dispensariam aplicativo, bateria e
cooperação. Ubiquidade máxima, consentimento zero. O grupo optou deliberadamente por não
maximizar a ubiquidade.

## Organização do repositório

| Pasta / arquivo | Conteúdo |
|---|---|
| [`marco-03/`](marco-03/README.md) | **Marco 3 — Segurança.** Canal cifrado (TLS), autenticação MQTT, todo dispositivo provando quem é (assinatura Ed25519 + token HMAC da âncora), atuação e produtor ↔ consumidor nos dois sentidos. Âncora Bluetooth real, app do celular, servidor e testes de ataque e de carga |
| [`marco-02/`](marco-02/README.md) | Marco 2 — fronteira de comunicação aparelho ↔ servidor por MQTT: produtor ESP32 (Wokwi), servidor Python, contrato versionado e condições de falha |
| [`docs/projeto-tecnico.md`](docs/projeto-tecnico.md) | Especificação completa: parâmetros, protocolo, máquina de estados, alternativas rejeitadas |
| [`docs/atividade-01.md`](docs/atividade-01.md) | Atividade em Grupo 01 — análise inicial (9 itens) |
| [`docs/atividade-02.md`](docs/atividade-02.md) | Atividade em Grupo 02 — processamento e distribuição (13 itens) |

Cada marco tem o seu próprio README com o que foi pedido, como foi feito, como rodar e como testar.

## Estado atual

| Etapa | Situação |
|---|---|
| Atividades 01 e 02 | análise concluída |
| Marco 2 | fronteira aparelho ↔ servidor por MQTT, com contrato e falhas (ESP32 no Wokwi) |
| Marco 3 | sistema com rádio de verdade (notebook como âncora Bluetooth, app Android) e segurança de ponta a ponta |

Próximos passos:

- [ ] Três âncoras ESP32 físicas com malha ESP-NOW (o firmware de uma âncora já está em `marco-03/esp32-ancora/`)
- [ ] Medição real da distância pelo sinal e ajuste do modelo
- [ ] Restrição de tópicos por usuário (ACL) no broker
- [ ] Painel do professor
