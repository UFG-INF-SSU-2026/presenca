# Presença — Documento de Projeto Técnico

Especificação de referência do sistema. Registra **o que está construído, as decisões
fechadas, os parâmetros numéricos e as alternativas rejeitadas**, para que não seja
necessário reconstruir o raciocínio a cada retomada.

Documentos irmãos: [`../README.md`](../README.md) (visão geral),
[`../marco-03/README.md`](../marco-03/README.md) (sistema atual, com segurança) e
[`atividade-01.md`](atividade-01.md) (análise acadêmica dos 9 itens).

> **Como ler este documento.** Desde o Marco 3, o sistema oficial é o protótipo com rádio de
> verdade: **um notebook como âncora Bluetooth, o celular Android do estudante medindo o sinal
> e um servidor Python de permanência**, conversando por MQTT cifrado. Cada seção descreve
> primeiro **o que existe e roda** e, quando é diferente, **o desenho para a sala real**
> (três âncoras, trilateração), marcado como *visão*. O que mudou em relação à proposta das
> Atividades 01 e 02 está resumido na [seção 13](#13-o-que-mudou-desde-a-proposta).

---

## 1. Escopo

O sistema mede a **permanência física** de estudantes em uma sala durante a janela da
aula e entrega ao professor a lista de frequência preenchida, mais avisos ao estudante
quando uma ausência em curso ameaça sua presença.

### Dentro do escopo atual

| Item | Situação |
|---|---|
| Âncora Bluetooth anunciando identificador + token rotativo assinado (HMAC) | ✅ notebook (`marco-03/ancora/`) |
| App do estudante (Android) medindo o RSSI da âncora e enviando heartbeat assinado | ✅ `marco-03/app/` |
| Área de administrador no app: turma, grade de horários, código/QR, calibração a 1 m, aula de demonstração | ✅ `marco-03/app/` |
| Servidor de permanência: validação, máquina de estados, orçamento de ausência, avisos | ✅ `marco-03/servidor/` |
| Segurança de ponta a ponta: TLS, usuário e senha por papel, assinatura Ed25519 em toda mensagem | ✅ broker EMQX Cloud |
| Atuação no celular (vibração, notificação, aviso na tela) só com mensagem autêntica | ✅ |
| Contrato de mensagens versionado (JSON Schema) | ✅ `marco-03/contrato/v1/` |
| Testes automáticos, demonstração de ataques e teste de carga | ✅ 49 (servidor) + 22 (app) |

### Visão, ainda não construída

- Três âncoras ESP32 fixas na sala, com malha ESP-NOW e trilateração + polígono. O servidor
  já sabe trilaterar (exercitado no Marco 2); o firmware de uma âncora ESP32 está pronto em
  `marco-03/esp32-ancora/`, mas não foi gravado em placa.
- Painel web do professor e integração com o sistema acadêmico.
- Restrição de tópicos por usuário (ACL) no broker.

### Fora do escopo

Climatização, iluminação, consumo de energia, qualidade do ar. O sistema é de presença;
ampliar dobraria o escopo sem fortalecer a proposta. A âncora com sensor de CO₂ do Marco 2
(conferir a contagem do rádio contra a ocupação da sala) foi um experimento contra o conluio
em massa e **não faz parte do sistema atual**. iOS também ficou fora: o app é Android.

## 2. Arquitetura

### As quatro peças

| Peça | Implementação atual | Responsabilidade |
|---|---|---|
| **Âncora** | Notebook Windows, `ancora.py` (Bluetooth LE via WinRT) | Anunciar `PRS1` + número da âncora + token HMAC renovado a cada 12 s. Só transmite |
| **Celular do estudante** | App React Native (Expo SDK 57), APK Android | Ouvir a âncora, filtrar o RSSI, decidir a região de rádio, assinar e enviar o heartbeat; conferir e exibir o estado; atuar nos avisos |
| **Broker** | EMQX Cloud Serverless (MQTT 3.1.1) | Entregar mensagens; só aceita TLS e usuário/senha |
| **Servidor** | Python, `servidor.py` + `permanencia.py` + `seguranca.py` | Conferir assinatura, contrato, identidade e token; calcular a permanência; responder estado e avisos assinados |

O **celular do administrador** é o mesmo app, em outro papel: cria a turma, publica a grade
(assinada) e calibra o limiar de 1 m.

```
Âncora ──Bluetooth (anúncio)──▶ Celular ──MQTT/TLS──▶ Broker ──▶ Servidor
                                   ▲                                  │
                                   └──── estado e avisos assinados ◀──┘
```

### Direção do rádio — decisão estrutural

A âncora **transmite e nunca escuta** aparelhos. Ela é um farol. Consequências:

- A âncora nunca conhece a identidade de ninguém. Desligada e com a memória extraída, não
  revela dado pessoal.
- Quem mede o sinal é o celular. Um anúncio é ouvido por todos os celulares ao mesmo
  tempo: **não há limite de celulares por âncora**.
- A identidade trafega apenas entre celular e servidor, por canal cifrado.

### Produtor e consumidor nos dois sentidos

| | Produz | Consome |
|---|---|---|
| **Celular** | heartbeat (assinado, com o token da âncora) | estado e avisos (confere a assinatura do servidor) |
| **Servidor** | estado e avisos (assinados) | heartbeat (confere assinatura, identidade e token) |
| **Admin** | turma (assinada) | — |

### Visão: malha entre âncoras

Na sala real, três âncoras ESP32 se comunicariam por **ESP-NOW** e elegeriam uma líder, única
a falar com o broker. Reduz a carga no ponto de acesso a um terço, permite operar com Wi-Fi
fraco e reorganiza automaticamente quando a líder falha. Não implementado.

## 3. Identidade e segurança

Quatro jeitos de enganar o sistema, cada um respondido por uma camada:

| Ataque | Camada | Como |
|---|---|---|
| **Ler** as mensagens no caminho | TLS | broker só aceita 8883 (MQTT/TLS) e 8084 (`wss://`) |
| **Entrar** no broker sem ser do sistema | autenticação MQTT | um usuário e senha por papel: `presenca-servidor`, `presenca-app` |
| **Falar em nome de outro** | assinatura por dispositivo | envelope `seguranca.envelope.v1`, Ed25519 sobre `tópico + carga` |
| Dizer que **está na sala estando em casa** | token da âncora | HMAC que só vale na janela de 12 s atual |

A assinatura é necessária mesmo com senha porque a senha do app vai **dentro do APK**: quem o
descompilar a obtém. A senha separa "de dentro" de "de fora"; a assinatura separa um estudante
do outro.

### Identidade do estudante

- Na primeira abertura, o app gera uma chave **Ed25519** a partir de 32 bytes aleatórios e a
  guarda no armazenamento seguro do Android (`expo-secure-store`, cifrado pelo Keystore).
  A chave privada nunca sai do aparelho.
- O **pseudônimo é o hash da chave pública**: `aluno = "tok-" + SHA-256(chave)[0..8]`.
  Não há cadastro: o servidor recalcula o hash da chave que veio no envelope.
- Nunca se usa MAC, IMEI ou nome.

### Identidade dos outros dispositivos

| Dispositivo | Como prova quem é | Quem confere | Se falhar |
|---|---|---|---|
| Celular do estudante | assina cada heartbeat; o nome é o hash da chave | servidor | `SEM ASSINATURA`, `ASSINATURA FALSA`, `IDENTIDADE TROCADA` |
| Âncora | token HMAC que só ela e o servidor sabem gerar | servidor | `TOKEN VENCIDO` |
| Servidor | assina estado e avisos; a chave pública vem fixada no app | celular | mensagem ignorada e contada como falsa |
| Administrador | assina a turma; o estudante guarda a chave no pareamento | celular do estudante | *"Turma recusada"* |

### Pareamento

O estudante entra na turma **uma vez**, digitando o código de 6 caracteres ou lendo o QR que o
administrador mostra na primeira aula. Nesse momento o app guarda a chave pública do
administrador (confiança no primeiro uso); daí em diante só aceita turma assinada por ela.
Depois disso, nenhuma ação do estudante por aula.

### Prova de presença — o token da âncora

```text
token   = HMAC-SHA256( segredo da âncora , "anc-pc-1|" + ⌊hora ÷ 12 s⌋ )[0..3]
anúncio = "PRS1" | nº da âncora | token (4 bytes)     — empresa 0xFFFF (reservada para testes)
```

Só a âncora e o servidor têm o segredo e usam o mesmo relógio, então calculam o mesmo token
sem conversar. O celular só **repassa** o que ouviu; não consegue inventar. O servidor aceita
a janela atual, a anterior e a seguinte (folga de relógio); para heartbeat que esperou na fila
durante queda de rede, até 4 janelas para trás (48 s).

### Os segredos

Ficam em `segredos.json` (fora do git). `python configurar.py` gera o segredo da âncora, a
chave do servidor e o `app/src/lib/segredos.ts` que vai dentro do APK. Trocou senha ou chave:
rodar de novo e gerar APK novo.

### Ameaças e tratamento

| Ameaça | Tratamento | Situação |
|---|---|---|
| Ler o tráfego no Wi-Fi da faculdade | TLS obrigatório | ✅ testado |
| Conectar no broker sem senha | autenticação por papel | ✅ testado |
| Heartbeat sem assinatura, ou assinado dizendo ser outro aluno | envelope + pseudônimo = hash da chave | ✅ testado |
| Mover uma mensagem válida para o tópico de outro aluno | a assinatura cobre o tópico | ✅ testado |
| Forjar heartbeat de casa | token da âncora | ✅ testado |
| Reusar captura de outro momento | token vence em 12 s | ✅ testado |
| Estado ou aviso falso para o celular | assinatura do servidor, chave fixada no app | ✅ testado |
| Turma falsa (ex.: limiar de 1 m em −100 dBm) | assinatura do administrador | ✅ testado |
| Senha do app extraída do APK | publica, mas é recusado; ainda lê pseudônimos | ⚠️ falta ACL |
| Celular com root extraindo a chave | chave fica no armazenamento seguro, não dentro do chip | ⚠️ limite declarado |
| Relay do token por alguém na sala para quem está fora | janela de 12 s torna manual e trabalhoso | ⚠️ parcial |
| Emprestar o aparelho ao colega | revisão do professor | ⚠️ parcial |
| Aparelho pareado abandonado na sala | regra "uma chave ativa por estudante" | ❌ não implementada |
| Conluio em massa (todos deixam o celular) | sem defesa técnica no escopo atual | ❌ residual |

## 4. Protocolo

Contratos em [`marco-03/contrato/v1/`](../marco-03/contrato/v1/), validados pelo servidor com
JSON Schema. Campo desconhecido é rejeitado.

| Mensagem | De → para | Tópico (`ufg/ssu/presenca/alany/m3/…`) |
|---|---|---|
| `presenca.heartbeat.v2` | celular → servidor | `sala-pc/heartbeat` |
| `presenca.estado.v1` | servidor → celular (retida) | `sala-pc/estado/<aluno>` |
| `presenca.aviso.v1` | servidor → celular | `sala-pc/aviso/<aluno>` |
| `presenca.turma.v1` | admin → celulares (retida) | `turma/<código>` |

Toda mensagem viaja dentro de um `seguranca.envelope.v1`:

```json
{
  "eventType": "seguranca.envelope.v1",
  "de": "tok-10ba682c",
  "chavePublica": "d04ab232…8737",
  "carga": "{\"eventType\":\"presenca.heartbeat.v2\", …}",
  "assinatura": "d102ad2b…a405"
}
```

### Heartbeat (carga)

```json
{
  "eventType": "presenca.heartbeat.v2",
  "eventId": "cel-8ad1#<bootId>#42",
  "deviceId": "cel-8ad1",
  "entityId": "sala-pc",
  "aluno": "tok-10ba682c",
  "bootId": "<sorteado a cada abertura do app>",
  "eventTimeMs": 1760600000000,
  "sequence": 42,
  "value": -58.5,
  "unit": "dBm",
  "state": "DENTRO_ALCANCE",
  "ancoras": [ { "id": "anc-pc-1", "rssi": -58.5, "token": "45cb0d0e" } ],
  "medidoEmMs": 1760600000000
}
```

`eventId = deviceId#bootId#sequence` permite ao servidor deduplicar, recolocar na ordem o que
chegou atrasado e contar o que se perdeu. O `bootId` existe porque reiniciar o app zera a
sequência. O carimbo de tempo que **vale** é o do servidor; o do celular é informativo.

O heartbeat leva até 3 âncoras, cada uma com o seu token, e o servidor confere cada token
contra o segredo daquela âncora: está pronto para a sala com três âncoras.

### Cadência

| | Projeto (sala real) | Demonstração |
|---|---|---|
| Heartbeat | 60–90 s | 500 ms |
| Amostra de RSSI | contínua | mediana dos últimos 2 s |
| Âncora sumiu | — | 4 s sem anúncio |

A permanência é derivada do **intervalo entre eventos**, não de uma contagem de amostras.
Com a rede fora, o app retém até 60 heartbeats (30 s) e os envia marcados como `bufferizado`.

### Sessão e janela da aula

O app só mede e envia **durante uma aula**: um período da grade da turma (dia, início e fim)
ou uma aula de demonstração iniciada pelo administrador agora. Fora disso, fica inerte — é o
duty cycling que importa, porque a bateria finita é a do celular. A âncora fica ligada o
tempo todo.

O servidor mantém **uma sessão por aluno**, aberta no primeiro heartbeat. Um boot novo no
meio da aula continua a mesma sessão; um boot novo depois de a aula terminar abre a seguinte.

## 5. Localização

### Hoje: uma âncora, limiar calibrado a 1 m

Com uma única âncora não há trilateração. A regra é **"na sala" = até 1 m do notebook**:

1. O administrador põe o celular a 1 m da âncora e calibra (8 s de medição). O RSSI medido
   vira o `limiar1mDbm` da turma, publicado assinado.
2. O celular do estudante filtra o RSSI (mediana de 2 s, mínimo de 3 amostras) e decide a
   região com **histerese de 5 dB**: entra em `DENTRO_ALCANCE` acima do limiar e só volta a
   `FORA_ALCANCE` 5 dB abaixo. Sem calibração, a região fica `INDETERMINADA`.
3. O servidor aceita a região como a decisão inteira.

É o "raio" que o projeto rejeita para a sala real (seção 10). Aqui é a **simplificação
declarada do protótipo**, coerente com uma âncora só.

### Visão: três âncoras, trilateração e polígono

```
RSSI = A − 10 · n · log₁₀(d)        A = RSSI a 1 m (calibrado) · n ≈ 2,2 (estimado)
```

Três distâncias estimadas produzem uma posição, e o teste é **pertinência ao polígono da
sala**, não distância a um ponto. Um raio é um círculo, uma sala é um retângulo com paredes, e
rádio atravessa parede: apertando o raio perde-se quem está no fundo; abrindo, pega-se a sala
vizinha. Ajustar o número não converge.

O código já existe: `permanencia.py` trilatera e testa o polígono sempre que a sala tem mais
de uma âncora configurada (`config.json` → `sala.ancoras`, `sala.poligono`). Foi exercitado no
Marco 2 com RSSI sintético (três potenciômetros no Wokwi), incluindo o caso do sinal forte na
sala vizinha.

### Calibração

Os limiares são definidos **em dBm, medidos no local** — nunca em metros. Metro é apenas a forma
de explicar o limiar a uma pessoa. A medição da curva real distância × RSSI está pendente
(`marco-03/medicoes/distancia.csv` → `ajustar_distancia.py`).

## 6. Máquina de estados e regras de permanência

Toda a regra está em [`permanencia.py`](../marco-03/servidor/permanencia.py), um módulo sem
rede e sem relógio, coberto por testes.

### Classificação de cada heartbeat

| Região do celular | (visão) Posição | Classe |
|---|---|---|
| `FORA_ALCANCE` | — | fora |
| `DENTRO_ALCANCE` | dentro do polígono | dentro |
| `DENTRO_ALCANCE` | fora do polígono | fora (sinal forte na sala vizinha) |
| `INDETERMINADA` ou âncora faltando | — | neutro: não há base para decidir |

Com uma âncora só, a coluna de posição não se aplica: `DENTRO_ALCANCE` é dentro.

### Estados

| Estado | Relógio | Transição de saída |
|---|---|---|
| `FORA` | parado | → `PRESENTE` com "dentro" sustentado por 30 s |
| `PRESENTE` | **contando** | → `AUSENCIA_TOLERADA` com "fora" sustentado por 30 s |
| `AUSENCIA_TOLERADA` | contando provisoriamente | → `PRESENTE` se retornar enquanto houver orçamento; senão → `FORA`, estornando o tempo desde o início real da ausência |
| `PAUSA_PROGRAMADA` | congelado | *visão*: entra e sai pelo relógio da agenda (intervalo da aula) |

Neutro interrompe a contagem de persistência. A entrada e a saída valem **desde o início real**
da condição, não desde o momento em que ela foi confirmada.

### Parâmetros

Os valores são os reais do projeto e ficam em `servidor/config.json`; na demonstração são
divididos pela **escala** (padrão 30: 1 s de demonstração = 30 s de aula).

| Parâmetro | Projeto | Demonstração (÷30) | Razão |
|---|---|---|---|
| Persistência de entrada e saída | 30 s | 1 s | Evita oscilação na borda |
| **Orçamento de ausência creditada** | **15 min por sessão** | 30 s | Cobre banheiro, café e telefonema, e limita o total em vez do evento |
| Avisos ao estudante | 10 min (2/3) e 13 min (87 %) | 20 s e 26 s | Chegam enquanto ainda cabe reação |
| Requalificação no retorno | `min(30 min, restante da aula)` | `min(60 s, restante)` | Impede reabrir crédito aparecendo 20 s |
| Aula | 50 min | 100 s | |
| Aprovação por aula-hora | ≥ 75 % (37,5 min) | 75 % | Converte contínuo em booleano legal |

### Regras invioláveis

1. **O acumulador é monotônico.** Congela e retoma; nunca zera. Quem assistiu 90 de 100
   minutos e saiu no fim soma 90. O que ainda pode mudar aparece em separado, como
   `provisorio`.
2. **A permanência é recalculada sobre o log inteiro** a cada heartbeat, nunca incrementada ao
   vivo. Só se descobre que a ausência começou às 10h42 depois — e o crédito da requalificação
   só se resolve após o retorno sustentado.
3. **O orçamento é por sessão, não por evento.** Sem isso, sair repetidamente por 29 min sai de
   graça.
4. **Fato bruto e inferência são guardados separados.** O servidor grava cada heartbeat aceito
   com a classificação e o que derivou dele (`recebidos.log`): guardar o porquê da decisão é o
   que permite auditar a reclamação do estudante.
5. **Existe override manual do professor** (*visão*, com o painel). Todo sistema de inferência
   precisa de válvula de escape humana.

## 7. Notificações (atuação)

O aviso só **atua** se chegar enquanto ainda cabe reação. Notificação depois da falta
consolidada é informação; antes, é atuação — e é o que fecha o laço ciber-físico: o estudante
volta e altera a própria posição, que é a grandeza medida.

| Momento | Mensagem |
|---|---|
| 10 min de ausência | *"Fora da sala há 10 min. Seu limite é de 15 min nesta aula."* |
| 13 min | *"Faltam 2 min do seu limite. Depois disso o tempo para de contar."* |
| Limite esgotado | *"O tempo parou de ser contabilizado às 10h42."* |
| Fim da aula | *"Você acumulou 78 % desta aula."* |

Cada aviso tem `avisoId` único (o celular não repete) e vem **assinado pelo servidor**. O celular
só atua — vibra, mostra notificação local e o cartão na tela — se a assinatura conferir com a
chave do servidor fixada no app. Nada é notificado fora desses quatro momentos.

## 8. Modelo de dados

### Hoje

| Onde | O que guarda |
|---|---|
| Celular (SQLite) | turmas (código, nome, sala, grade, limiar de 1 m, chave do admin), aulas, avisos recebidos, ajustes |
| Celular (armazenamento seguro) | chave Ed25519 do aparelho, PIN do administrador |
| Broker (mensagens retidas) | a turma publicada; o último estado de cada aluno |
| Servidor (`recebidos.log`, JSON por linha) | cada heartbeat aceito, com carimbo do servidor, sessão, classe e derivado |
| Servidor (memória) | sessões por aluno, eventos já vistos |

### Visão (painel do professor)

| Entidade | Campos que importam |
|---|---|
| `Sala` | identificadores das âncoras, polígono |
| `Turma` / `AulaAgendada` | vínculo professor ↔ estudantes; dia, início, fim, intervalo, sala |
| `SessaoDeAula` | instância real do dia (pode ser cancelada ou mudar de sala) |
| `Matricula` | estudante ↔ turma, **uma** chave pública ativa |
| `EventoSensor` | bruto, retenção curta (dias) |
| `Presenca` | derivado: estado final, janelas de ausência, **como foi decidido** |

**Nunca materializar nem persistir o grafo de colocalização** (quem esteve perto de quem). É o
dado mais sensível que o sistema obteria de graça, e ele não é necessário.

## 9. Restrições de plataforma

- O app do estudante **precisa ser nativo**: não existe API de RSSI em navegador algum. Por
  isso ele roda como **APK**, não no Expo Go (que não tem Bluetooth).
- O Android esconde anúncios de farol de apps que declaram não usar Bluetooth para localização;
  o plugin `ble-sem-never-for-location.js` ajusta isso.
- O JavaScript pausa em segundo plano: durante a aula a tela fica acesa. Na versão de
  produção, um serviço em primeiro plano (Android) faria esse papel.
- Âncora e servidor precisam de relógio certo para o token (hoje são o mesmo notebook; no
  ESP32, NTP).
- O broker EMQX gratuito deixa passar **~11 mensagens por segundo por conexão**. O servidor
  sozinho processa ~2.400 msg/s. Com heartbeat de 60 s numa aula real, ≈ 650 celulares por
  conexão do servidor; acima disso, assinatura compartilhada (`$share/…`) com várias cópias do
  servidor.
- O **painel do professor seria web**: é interativo, em primeiro plano, e não sensoria nada.
  Quem sensoria precisa de nativo, quem consulta não.

## 10. Alternativas rejeitadas

Registradas para não serem re-propostas.

| Alternativa | Por que foi rejeitada |
|---|---|
| **Celular do professor varrendo os alunos** | Não escala (1 aparelho × N alunos), consome a bateria do dispositivo crítico, e o iOS restringe o scan em segundo plano |
| **Âncora escutando os celulares** | A âncora passaria a saber quem está perto. `escutar.py` existe só como medição, fora do sistema |
| **Endereço MAC como identidade** | Randomizado por rede desde Android 10 / iOS 14; IMEI inacessível; ID de anúncio zerável |
| **Identidade por cadastro no servidor** | Desnecessária: o pseudônimo é o hash da chave pública e o servidor o recalcula |
| **GPS / localização do sistema** | 10–50 m de erro dentro de prédio. Não distingue sala de sala |
| **P2P puro (alunos atestando uns aos outros)** | A turma inteira tem incentivo alinhado para mentir. Viável apenas ancorado |
| **QR a cada aula** | Atrito em 100 % das aulas para pegar fraude rara. Substituído por pareamento único + token da âncora |
| **Limiar em metros como regra da sala real** | RSSI não mede distância: ±5–10 m de erro. O problema é círculo × polígono. (Usado hoje só porque há uma âncora) |
| **Wi-Fi do celular como sinal** | A força satura em torno de −55 dBm perto do roteador: não distingue 1 m de 5 m |
| **ESP32 simulando o celular do estudante** | Usado no Marco 2 (Wokwi). O professor pediu o celular de verdade; o ESP32 volta como âncora |
| **Broker público (`broker.hivemq.com`)** | Sem senha: qualquer um lê e publica. Demonstrado em `ataques.py publico` |
| **HiveMQ Cloud** | Deixou de ter plano gratuito. EMQX Cloud Serverless dá TLS, senha e ACL de graça, limite de gasto US$ 0 |
| **Só senha, sem assinatura** | A senha do app vai dentro do APK; não separa um estudante do outro |
| **Token da âncora sorteado** | Qualquer um inventaria. HMAC com segredo compartilhado com o servidor |
| **Zerar o acumulador ao sair** | Perverso: quem assistiu 90 de 100 min perderia tudo |
| **Tolerância por evento em vez de orçamento** | Explorável por ciclos: 30 min dentro / 29 min fora |
| **Sensores de CO₂, climatização e iluminação** | Dobram o escopo. A notificação ao estudante fecha o laço sem hardware novo. (CO₂ explorado no Marco 2, fora do sistema atual) |
| **Aplicativo web / PWA** | Sem API de RSSI em navegador |
| **802.1X / RADIUS (sem aplicativo)** | Sem software no aparelho, mas perde a precisão e depende da TI liberar os logs |

## 11. Questões em aberto

- [ ] Três âncoras físicas (ESP32) e a malha ESP-NOW; gravar `esp32-ancora.ino` em placa
- [ ] Medição real da curva distância × RSSI e ajuste do expoente
- [ ] ACL por tópico no EMQX (cada celular só publica no próprio tópico)
- [ ] Regra "uma chave ativa por estudante" (exige matrícula no servidor)
- [ ] Painel do professor, override manual e pausa programada
- [ ] Serviço em primeiro plano no Android, para medir com a tela apagada
- [ ] Relay do token e empréstimo de aparelho continuam parciais; conluio em massa, residual
- [ ] Integração com o sistema acadêmico da instituição

## 12. Ambiente de execução e testes

| Peça | Onde roda |
|---|---|
| Âncora | notebook com Bluetooth (`python ancora.py`) |
| Servidor | o mesmo notebook (`python servidor.py`, MQTT sobre `wss://` porta 8084) |
| Broker | EMQX Cloud Serverless |
| Estudante e administrador | celulares Android com o APK *Presença M3* |
| Muitos estudantes | `ataques/carga.py`: celulares simulados, cada um com a própria chave, assinando de verdade |

Redes que bloqueiam 8883/8084: hotspot do celular. `servidor/testar_rede.py` diz se a rede
alcança o broker.

**Tempos comprimidos.** Todos os limiares são constantes de configuração com o valor real,
divididos pela escala. A demonstração de 100 s equivale a uma aula de 50 min.

**Testes:** `python -m pytest testes -q` em `marco-03/servidor` (49) e `npm test` em
`marco-03/app` (22, incluindo a assinatura conferida entre JavaScript e Python). Ataques
demonstráveis: `python ataques.py senha | forjar | estado | turma | publico --ws`.

**Wokwi (Marco 2).** O Wokwi não simula propagação de rádio. No Marco 2 o aparelho do
estudante era um ESP32 com três potenciômetros fazendo o papel do RSSI das três âncoras — é onde
a trilateração e o polígono foram exercitados.

## 13. O que mudou desde a proposta

| Proposta (Atividades 01 e 02) | Hoje |
|---|---|
| 3 âncoras ESP32 com malha ESP-NOW | 1 âncora: o notebook (ESP32 pronto em código, sem placa) |
| Trilateração + polígono | Limiar calibrado a 1 m; trilateração pronta no servidor para 3 âncoras |
| Heartbeat por HTTPS | MQTT sobre TLS (EMQX Cloud) |
| Android e iOS | Android (APK) |
| Gateway de borda abrindo e encerrando sessões | A grade da turma (publicada pelo admin) define a janela; o servidor abre uma sessão por aluno |
| QR individual de uso único | Código/QR da turma + chave do aparelho; pseudônimo = hash da chave |
| Chave no Secure Enclave / Keystore | Chave Ed25519 no armazenamento seguro (cifrado pelo Keystore) |
| Bateria e acelerômetro no heartbeat | Fora do heartbeat atual |
| Token rotativo a cada 10–15 s com nonce da sessão | Token HMAC por janela de 12 s |
| Painel web do professor | Painel no terminal do servidor; web é visão |
