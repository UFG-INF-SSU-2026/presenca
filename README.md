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

## Como funciona hoje

![As quatro peças do sistema](marco-03/documentacao/imagens/arquitetura.svg)

1. **A âncora** — hoje, um notebook — anuncia por Bluetooth o seu identificador e um
   **token que muda a cada 12 s**, calculado com um segredo que só ela e o servidor
   conhecem. Ela só transmite: nunca sabe quem está na sala.
2. **O app do estudante** (Android) ouve a âncora, mede a intensidade do sinal e decide se
   está **a até 1 m** dela, contra um limiar que o administrador calibrou. A cada batida,
   envia ao servidor um heartbeat **assinado** com a chave do próprio aparelho, levando o
   token que acabou de ouvir.
3. **O servidor** confere assinatura, contrato, identidade e token; só então alimenta a
   máquina de estados que acumula o tempo de permanência, com um orçamento de 15 min de
   ausência por aula.
4. **Quando uma ausência chega a 10 e a 13 minutos**, o servidor manda um aviso assinado e
   o celular vibra e notifica — *"fora da sala há 10 min, seu limite é 15"*. É a
   notificação que fecha o laço: o estudante retorna e altera a própria posição, que é a
   grandeza medida.
5. **Ao fim da aula**, cada estudante tem o percentual acumulado; presença é ≥ 75 %.

Tudo passa por MQTT com **TLS e usuário/senha** (EMQX Cloud). O **administrador** usa o
mesmo app para criar a turma, definir a grade de horários, mostrar o código/QR de entrada
e calibrar o limiar de 1 m.

## Decisões de projeto

| Decisão | Motivo |
|---|---|
| **Permanência, não presença** | O erro do sistema passa a ser proporcional à falha: três minutos de sinal ruim custam três minutos, não a aula inteira |
| **Âncora só transmite, nunca escuta** | A infraestrutura da sala não guarda nenhum dado pessoal, e não há limite de celulares por âncora |
| **Token da âncora com HMAC** | Prova que o celular estava ouvindo a âncora *agora*. Heartbeat feito em casa ou repetido depois não tem o token certo |
| **Identidade por chave, não por hardware** | MAC é randomizado, IMEI é inacessível. O aparelho gera uma chave Ed25519 e o nome do estudante é o hash dela — sem cadastro, sem nome no tráfego |
| **Toda mensagem assinada** | A senha do app vai dentro do APK; ela separa quem é do sistema, a assinatura separa um estudante do outro. Servidor e admin também assinam |
| **Entrar na turma uma vez** | Código ou QR na primeira aula. Depois disso, nenhuma ação do estudante por aula |
| **Orçamento de ausência por sessão** | 15 minutos de tolerância no total da aula, não por evento — senão basta sair repetidamente por pouco tempo |
| **Avisar antes de punir** | Notificação após a falta consolidada é informação. Antes, é atuação |

Para a sala real, o desenho prevê **três âncoras ESP32** e **trilateração + polígono da
sala** em vez do limiar de 1 m — um raio é um círculo, uma sala é um polígono. O servidor
já trilatera quando há três âncoras; o firmware da âncora ESP32 está pronto, sem placa
para gravar. Detalhes em [`docs/projeto-tecnico.md`](docs/projeto-tecnico.md).

## Classificação

| Categoria | Situação |
|---|---|
| **Aplicação ubíqua** | ✅ principal — interação implícita, ciência de contexto, infraestrutura invisível |
| **Internet das Coisas** | ✅ dispositivos conectados, MQTT com TLS, processamento remoto |
| **Rede de sensores** | ✅ âncora + celulares medindo o sinal + servidor; uma âncora física hoje, três na visão |
| **Sistema ciber-físico** | ✅ com humano no laço — a notificação atua sobre a posição medida |

Justificativas completas em [`docs/atividade-01.md`](docs/atividade-01.md), item 8.

## Risco principal

**Privacidade.** Não pelo volume de dado, que é pequeno, mas por um subproduto que o
sistema obtém de graça sem precisar: o **grafo de colocalização** — quem senta perto de
quem ao longo do semestre. Ele nunca é materializado nem persistido.

O compromisso de fundo, registrado no documento: *neste domínio, ubiquidade e privacidade
são grandezas inversas*. As versões mais invisíveis do sistema — câmera com
reconhecimento facial, sensor de pressão no piso — dispensariam aplicativo, bateria e
cooperação. Ubiquidade máxima, consentimento zero. O projeto optou deliberadamente por não
maximizar a ubiquidade.

## Organização do repositório

| Pasta / arquivo | Conteúdo |
|---|---|
| [`marco-03/`](marco-03/README.md) | **Sistema atual (Marco 3 — Segurança).** Âncora Bluetooth (notebook), app Android do estudante e do admin, servidor de permanência, contrato, ataques e teste de carga. Canal cifrado, autenticação MQTT e todo dispositivo provando quem é |
| [`marco-02/`](marco-02/README.md) | Marco 2 — fronteira aparelho ↔ servidor por MQTT, com o aparelho simulado por um ESP32 no Wokwi (três potenciômetros no papel das três âncoras), contrato versionado e condições de falha |
| [`docs/projeto-tecnico.md`](docs/projeto-tecnico.md) | Especificação: o que está construído, parâmetros, protocolo, máquina de estados, segurança, alternativas rejeitadas e o que mudou desde a proposta |
| [`docs/atividade-01.md`](docs/atividade-01.md) | Atividade em Grupo 01 — análise inicial (9 itens) |
| [`docs/atividade-02.md`](docs/atividade-02.md) | Atividade em Grupo 02 — processamento e distribuição (13 itens) |

Cada marco tem o seu próprio README com o que foi pedido, como foi feito, como rodar e
como testar. Para rodar o sistema atual, veja
[`marco-03/README.md` → Como rodar](marco-03/README.md#como-rodar).

## Estado atual

| Etapa | Situação |
|---|---|
| Atividades 01 e 02 | análise concluída |
| Marco 2 | fronteira aparelho ↔ servidor por MQTT, com contrato e falhas (ESP32 no Wokwi) |
| Marco 3 | sistema com rádio de verdade (notebook como âncora, app Android) e segurança de ponta a ponta; 49 testes no servidor e 22 no app |

Próximos passos:

- [ ] Três âncoras ESP32 físicas com malha ESP-NOW e trilateração na sala real
- [ ] Medição real da distância pelo sinal e ajuste do modelo
- [ ] Restrição de tópicos por usuário (ACL) no broker
- [ ] Uma chave ativa por estudante
- [ ] Painel do professor, com revisão de exceções
