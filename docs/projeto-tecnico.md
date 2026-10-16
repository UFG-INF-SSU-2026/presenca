# Presença — Documento de Projeto Técnico

Especificação de referência do sistema. Registra **decisões fechadas, parâmetros
numéricos e alternativas rejeitadas**, para que não seja necessário reconstruir o
raciocínio a cada retomada.

Documentos irmãos: [`../README.md`](../README.md) (visão geral) e
[`atividade-01.md`](atividade-01.md) (análise acadêmica dos 9 itens).

---

## 1. Escopo

O sistema mede a **permanência física** de estudantes em uma sala durante a janela da
aula e entrega ao professor a lista de frequência preenchida, mais avisos ao estudante
quando uma ausência em curso ameaça sua presença.

**Fora de escopo:** climatização, iluminação, consumo de energia, qualidade do ar. Foram
considerados e descartados — o sistema é de presença, e ampliar dobraria o escopo sem
fortalecer a proposta.

## 2. Arquitetura

### Camadas

| Camada | Componente | Responsabilidade |
|---|---|---|
| Sala | 3 × âncora ESP32 | Transmitir sinal identificado + token rotativo; malha entre si |
| Sala | Smartphone do estudante | Medir sinal das âncoras, IMU e bateria; assinar e enviar heartbeat |
| Borda | Gateway | Agenda, abertura/encerramento de sessão, carimbo de tempo, amortecimento |
| Núcleo | Servidor | Trilateração, máquina de estados, banco, verificação criptográfica |
| Núcleo | Painel web | Consulta e revisão de exceções pelo professor |

### Direção do rádio — decisão estrutural

```
Âncora ──broadcast──▶ Smartphone ──HTTPS──▶ Servidor
```

A âncora **transmite e nunca escuta** aparelhos. Ela é um farol. Consequências:

- A âncora nunca conhece a identidade de ninguém. Arrancada da parede e com a memória
  extraída, não revela dado pessoal.
- Quem mede o sinal é o smartphone (N aparelhos ouvindo 3 faróis escala melhor que
  1 aparelho varrendo N).
- A identidade trafega apenas entre aparelho e servidor, por canal cifrado.

### Malha entre âncoras

As 3 âncoras da sala se comunicam por **ESP-NOW** e elegem uma líder, única a falar com
o gateway por MQTT. Reduz a carga no ponto de acesso a um terço, permite operar em sala
com Wi-Fi fraco, e reorganiza automaticamente quando a líder falha.

## 3. Identidade e segurança

### Pareamento — uma vez por semestre

1. O estudante lê um **QR individual de uso único**, entregue pelo portal acadêmico.
   Nunca um código único projetado para a turma — a foto circula.
2. O aplicativo gera um par de chaves **dentro do hardware seguro** (Secure Enclave no
   iOS, Keystore no Android). A chave privada nunca sai do cofre.
3. O servidor grava o vínculo `aluno_id ↔ chave_pública` e queima o token.

**Regra obrigatória: uma chave ativa por estudante.** Parear um aparelho novo revoga o
anterior. Sem isso, basta parear um aparelho barato e deixá-lo escondido na sala.

### Quando exige novo pareamento

| Evento | Repareia? |
|---|---|
| Reinício, atualização de app ou de sistema, troca de Wi-Fi ou de chip | Não |
| Desinstalação do app, restauração de fábrica, aparelho novo | Sim |
| Android: reset de biometria/PIN pode invalidar o Keystore | Às vezes |

### Prova de presença

A âncora transmite um **token rotativo** (HMAC sobre contador de tempo, renovado a cada
10–15 s). O aparelho só devolve o token válido se realmente o ouviu — prova de recepção,
logo de proximidade física. Exigir os tokens das **três** âncoras simultaneamente
reforça.

O heartbeat é **assinado** com a chave privada; o servidor verifica com a pública. O que
trafega é assinatura, não identificador permanente.

### Ameaças e tratamento

| Ameaça | Tratamento | Resolvido? |
|---|---|---|
| Forjar RSSI de casa | Token rotativo — impossível adivinhar | ✅ |
| Reusar captura de outro dia | Nonce da sessão no token | ✅ |
| Conluio em massa (todos deixam o celular) | Sem defesa técnica no escopo atual | ❌ residual |
| Relay do token por mensagem a quem está fora | Janela de 10–15 s torna manual e trabalhoso | ⚠️ parcial |
| Emprestar o aparelho ao colega | Variância de acelerômetro + revisão do professor | ⚠️ parcial |
| Mentir nível de bateria para forjar "morreu" | Coerência da curva de descarga (é lenta e monotônica) | ✅ |
| Aparelho pareado abandonado na sala | Uma chave ativa por estudante + detecção de imobilidade | ✅ |

## 4. Protocolo

### Heartbeat do aparelho → servidor

```json
{
  "sessao": "<nonce da aula>",
  "aluno": "<token>",
  "ts": 1758000000,
  "bateria": { "nivel": 63, "carregando": false },
  "imu": { "variancia": 0.042 },
  "ancoras": [
    { "id": "esp32-204-A", "rssi": -68, "token": "<token rotativo ouvido>" },
    { "id": "esp32-204-B", "rssi": -81, "token": "..." },
    { "id": "esp32-204-C", "rssi": -77, "token": "..." }
  ]
}
```

Assinado com a chave privada do aparelho. O carimbo de tempo que **vale** é o do
servidor; `ts` do aparelho é informativo e nunca fonte de verdade.

### Cadência

| Plataforma | Mecanismo | Cadência |
|---|---|---|
| **Android** | Foreground Service com notificação persistente | 60–90 s |
| **iOS** | Monitoramento de região (o sistema acorda o app na entrada e na saída) | Orientado a evento |

Consequência de projeto: a permanência é derivada do **intervalo entre eventos**, não de
uma contagem de amostras periódicas. Só assim os dois sistemas convergem.

### Sessão

O gateway publica `sessao/iniciar` 5 minutos antes da aula e `sessao/encerrar` ao fim.
A âncora **permanece energizada e conectada** o tempo todo — apenas o sensoriamento é
que liga e desliga. Motivos: mudança de agenda propaga na hora, sem latência de boot
(2–10 s para reassociar Wi-Fi), e âncora quebrada é detectada com antecedência.

O duty cycling que importa é no aparelho do estudante, onde a bateria é finita: o app
fica inerte fora da janela `[início − 5 min, fim]`.

## 5. Localização

### Modelo de propagação

```
RSSI = A − 10 · n · log₁₀(d)
```

`A` ≈ −45 dBm a 1 m; `n` entre 2 e 4 conforme obstrução. Usado no **simulador**; na
operação real o caminho é inverso (RSSI → distância estimada).

### Trilateração e polígono

Três distâncias estimadas produzem uma posição. O teste é **pertinência ao polígono da
sala**, não distância a um ponto.

> Razão da decisão: um raio é um círculo, uma sala é um retângulo com paredes, e rádio
> atravessa parede. Não existe valor de raio que coincida com a sala — apertando, perde
> quem está no fundo; abrindo, pega a sala vizinha. Ajustar o número não converge.

### Calibração

Os limiares são definidos **em dBm, obtidos por levantamento na sala real** (quatro
cantos, porta, corredor, sala vizinha) — nunca em metros. Metro é apenas a forma de
explicar o limiar a uma pessoa.

## 6. Máquina de estados e regras de permanência

### Estados

| Estado | Relógio | Transição de saída |
|---|---|---|
| `FORA` | parado | → `PRESENTE` com RSSI > −70 dBm por 30 s |
| `PRESENTE` | **contando** | → `AUSENCIA_TOLERADA` com RSSI < −85 dBm por 30 s |
| `AUSENCIA_TOLERADA` | contando provisoriamente | → `PRESENTE` se retornar enquanto houver orçamento; senão → `FORA`, estornando o tempo desde o início real da ausência |
| `PAUSA_PROGRAMADA` | congelado | Entra e sai pelo relógio da agenda, independente de sinal |

A dupla de limiares (−70 / −85 dBm) é **histerese**: a faixa morta entre eles impede
oscilação de quem está sentado na borda do alcance.

`motivo` é **atributo do evento de ausência**, não estado: `saiu | bateria | app_morto |
desconhecido`. Decide se a ausência consome orçamento, vai para revisão, ou pune.

### Parâmetros

| Parâmetro | Valor | Razão |
|---|---|---|
| Limiar de entrada | > −70 dBm por 30 s | Histerese |
| Limiar de saída | < −85 dBm por 30 s | Histerese |
| **Orçamento de ausência creditada** | **15 min por sessão** | Limite único. Cobre banheiro, café e telefonema, e limita o total em vez do evento |
| Avisos ao estudante | 10 min (2/3) e 13 min (87 %) | Chegam enquanto ainda cabe reação |
| Requalificação no retorno | `min(30 min, tempo restante da aula)` | Impede reabrir crédito aparecendo 20 s |
| Pausa programada | Da agenda (+5 min de margem) | Congela para a turma inteira |
| Aprovação por aula-hora | ≥ 75 % de 50 min = 37,5 min | Converte contínuo em booleano legal |

### Regras invioláveis

1. **O acumulador é monotônico.** Congela e retoma; nunca zera. Quem assistiu 90 de 100
   minutos e saiu no fim soma 90.
2. **A permanência é calculada retroativamente sobre o log**, nunca incrementada ao
   vivo. Só se descobre que a ausência começou às 10h42 dez minutos depois — e o crédito
   da requalificação só se resolve 30 minutos após o retorno.
3. **O orçamento é por sessão, não por evento.** Sem isso, sair repetidamente por 29 min
   sai de graça: 60 min reais rendem 89 % de presença computada.
4. **`EventoSensor` e `Presenca` são tabelas distintas.** A primeira é fato bruto; a
   segunda é inferência, e inferência erra. Guardar o porquê da decisão é o que permite
   auditar a reclamação do estudante.
5. **Existe override manual do professor.** Todo sistema de inferência precisa de válvula
   de escape humana, ou um erro de RSSI vira reprovação por falta.

## 7. Notificações

O aviso só **atua** se chegar enquanto ainda cabe reação. Notificação após a falta
consolidada é informação; antes, é atuação — e é o que fecha o laço ciber-físico.

| Momento | Mensagem |
|---|---|
| 10 min de ausência | *"Fora da sala há 10 min. Seu limite é de 15 min nesta aula."* |
| 13 min | *"Faltam 2 min do seu limite. Depois disso o tempo para de contar."* |
| Limite esgotado | *"O tempo parou de ser contabilizado às 10h42."* |
| Fim da aula | *"Você acumulou 78 % desta aula."* |

Nada é notificado fora desses quatro momentos. Se o estudante ignorar o aviso, o sistema
não insiste — insistir é ruído, e ele não tem autoridade sobre a decisão dele.

## 8. Modelo de dados

| Entidade | Campos que importam |
|---|---|
| `Sala` | Identificadores das 3 âncoras, polígono, BSSIDs que a cobrem |
| `Disciplina` / `Turma` | Vínculo professor ↔ estudantes |
| `AulaAgendada` | Dia, início, fim, **início e fim do intervalo**, sala |
| `SessaoDeAula` | Instância real do dia (pode ser cancelada ou mudar de sala) |
| `Matricula` | Estudante ↔ turma, chave pública ativa |
| `EventoSensor` | Bruto: timestamp, âncora, identificador, RSSI, origem. **Retenção curta (dias)** |
| `Presenca` | Derivado: estado final, janelas de ausência, motivo, **como foi decidido** |

**Nunca materializar nem persistir o grafo de colocalização** (quem esteve perto de
quem). É o dado mais sensível que o sistema obteria de graça, e ele não é necessário.

## 9. Restrições de plataforma

O aplicativo do estudante **precisa ser nativo**. Navegador não serve — e o motivo
principal encerra o assunto sozinho: **não existe API de RSSI ou BSSID em nenhum
navegador**. Somam-se: Web Bluetooth exige gesto por conexão e não existe no iOS; página
em segundo plano não executa; WebAuthn exige gesto do usuário a cada assinatura.

O **painel do professor é web**, e deve ser: é interativo, em primeiro plano, e não
sensoria nada. A assimetria é limpa — quem sensoria precisa de nativo, quem consulta não.

## 10. Alternativas rejeitadas

Registradas para não serem re-propostas.

| Alternativa | Por que foi rejeitada |
|---|---|
| **Celular do professor varrendo os alunos** | Não escala (1 aparelho × N alunos), consome a bateria do dispositivo crítico, e o iOS restringe severamente o scan em segundo plano |
| **Endereço MAC como identidade** | Randomizado por SSID desde Android 10 / iOS 14; muda ao esquecer a rede; IMEI inacessível; ID de anúncio zerável pelo usuário |
| **GPS / localização do sistema operacional** | 10–50 m de erro dentro de prédio, quando fixa. Não distingue sala de sala. Background estrangulado nas duas plataformas |
| **P2P puro (alunos atestando uns aos outros)** | A turma inteira tem incentivo alinhado para mentir — a multidão é o adversário. Além de vulnerável a Sybil sem raiz de confiança. Viável apenas **ancorado**, com 1 salto |
| **QR a cada aula** | Atrito em 100 % das aulas para pegar fraude rara, e previsível — o estudante organiza a fraude em torno do horário. Substituído por pareamento único + auditoria |
| **Limiar em metros (15 m, 20 m…)** | RSSI não mede distância: ±5–10 m de erro, variando com bolso, corpo e orientação. Ajustar o número não converge; o problema é círculo × polígono |
| **Detectar bateria morta pelo formato da queda de RSSI** | Sair de uma sala leva ~10 s; com amostragem de 60–90 s não se captura curva alguma. Substituído por nível de bateria no heartbeat, que é trivial e confiável |
| **Zerar o acumulador ao sair** | Perverso: quem assistiu 90 de 100 min perderia tudo. O acumulador pausa |
| **Tolerância por evento em vez de orçamento** | Explorável por ciclos: 30 min dentro / 29 min fora, repetidamente |
| **Sensores de CO₂, climatização e iluminação** | Fechariam o laço ciber-físico por atuação física, mas dobram o escopo. A notificação ao estudante fecha o mesmo laço sem hardware novo |
| **Aplicativo web / PWA** | Sem API de RSSI ou BSSID em navegador algum |
| **802.1X / RADIUS (sem aplicativo)** | Rota legítima e sem software no aparelho, mas perde a precisão do polígono, o IMU e o controle da amostragem, e depende da TI liberar os logs |

## 11. Questões em aberto

- [ ] Calibração empírica dos limiares em dBm numa sala real
- [ ] Verificar se o Wokwi suporta BLE no ESP32 de forma utilizável (suporte limitado)
- [ ] Ataque de relay do token permanece residual
- [ ] Empréstimo de aparelho é mitigado, não resolvido
- [ ] Integração com o sistema acadêmico da instituição
- [ ] Definir se o painel do professor exibe permanência contínua ou já discretizada

## 12. Ambiente de simulação

O Wokwi **não simula propagação de rádio** — não há RSSI, nem distância, nem celular.
Simula o chip, os periféricos e a rede (via `Wokwi-GUEST`, com internet real, o que faz
MQTT funcionar).

Divisão consequente:

| Peça | Onde |
|---|---|
| Firmware da âncora: beacon, token rotativo, malha, saúde | Wokwi (um projeto por âncora, mesmo broker) |
| Estudante emitindo heartbeat com RSSI sintético | Simulador em software — permite 40 estudantes com trajetórias e ruído |
| Trilateração, polígono, máquina de estados | Backend |

**Bancada de demonstração:** três potenciômetros nas ADCs da âncora, cada um simulando
um RSSI. Gira-se o botão e o painel mostra o ponto atravessar a parede e o relógio de
presença congelar.

**Armadilha:** o relógio do Wokwi não anda na velocidade real e oscila com a carga.
Todos os limiares devem ser constantes de configuração, testados com valores comprimidos
(30 min → 30 s).
