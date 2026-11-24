# sonoff-neway-automacoes

Nós para o Node-RED que controlam aparelhos Sonoff com o firmware original da eWeLink: interruptores, tomadas, relés e outros. Funcionam pela **rede local** (estado por anúncio mDNS, comandos por HTTP direto ao aparelho), pela **nuvem da eWeLink**, ou pelos dois (modo Automático). O login é só com e-mail e senha da conta eWeLink, como a integração SonoffLAN do Home Assistant.

Neway Automações.

Aviso: este pacote usa o mesmo acesso de aplicativo que o SonoffLAN usa. Não é uma API pública com garantia. Logar na nuvem com o mesmo usuário em outro lugar (por exemplo, no Home Assistant) pode derrubar a outra sessão.

Dependências: `bonjour-service` e `ws` (instaladas junto com o pacote).

## O que vem no pacote

Na paleta, no grupo **sonoff-neway-automacoes**, aparece um nó:

- **Sonoff**: lê o estado, consulta e envia comandos a um aparelho.

A conta eWeLink não aparece na paleta, porque é um nó de configuração. Ela é criada de dentro do nó **Sonoff** (campo **Conta**, botão do lápis) e fica listada no menu, em **Configuration nodes**.

## Instalar

A instalação é feita dentro do próprio Node-RED.

1. Baixe o arquivo `.tgz` deste repositório.
2. No Node-RED, abra o menu (três barras), **Gerenciar paleta**, aba **Instalar**, e envie o arquivo `.tgz` (botão de enviar/upload).
3. Confirme a instalação. Na paleta aparece o grupo **sonoff-neway-automacoes**. Se não aparecer, reinicie o Node-RED.

## Conta (feita uma vez)

1. Arraste um nó **Sonoff** para o fluxo, dê duplo clique e, no campo **Conta**, clique no lápis.
2. Preencha:

| Campo | O que é |
|---|---|
| Nome | qualquer nome, só para você reconhecer |
| E-mail eWeLink | o e-mail (ou telefone) da sua conta no app eWeLink; serve só para entrar na eWeLink |
| Senha | a senha da sua conta eWeLink; fica guardada de forma criptografada no Node-RED |
| Código do país | o código do seu país com `+`, por exemplo `+55` para o Brasil; a região da nuvem é descoberta sozinha |
| Modo | como o nó fala com os aparelhos: Automático, Local ou Nuvem (explicado abaixo) |

3. Faça o **Deploy**. A etiqueta da conta mostra `N/M online` quando está funcionando.

Na conta também há o botão **Atualizar lista** (campo "Nuvem"). Ele refaz o login e baixa a lista de aparelhos de novo. A sessão fica salva e é reaproveitada: reiniciar o Node-RED não refaz o login.

### Modos

- **Automático**: usa a rede local quando o aparelho é encontrado e a nuvem nos outros casos. Os comandos vão primeiro pela rede local e caem para a nuvem se falharem.
- **Local**: só rede local (mDNS). Não precisa de internet para controlar, mas a lista de aparelhos e as chaves vêm da nuvem: o primeiro login e o botão **Atualizar lista** precisam de internet.
- **Nuvem**: só a nuvem da eWeLink. Serve para aparelhos que estão em outra rede.

Um aparelho conta como online se estiver online pela rede local ou pela nuvem.

## Usar o nó Sonoff

| Campo | O que faz |
|---|---|
| Nome | só um rótulo para você reconhecer o nó no fluxo (opcional) |
| Conta | a conta eWeLink criada acima |
| Tipo | define o que o nó faz e quais portas tem: Status, Leitura ou Comando (explicados abaixo) |
| Dispositivo | o aparelho, escolhido numa lista com os aparelhos da conta ("- offline" aparece nos que não estão ativos) |
| Código | opcional. O ponto de dados do aparelho a usar (por exemplo `switch`, `switch_1`). A lista mostra os códigos do aparelho escolhido. Em "Todos os códigos", o nó trata todos |
| Emitir ao iniciar | só no tipo Status. Marcado: assim que a conta carrega, o estado atual sai uma vez, sem esperar uma mudança |
| Comando, Ter saída, Atraso, Passar adiante | só no tipo Comando (explicados abaixo) |

### Tipo: Status (só saída)

Emite uma mensagem por código que mudou, em tempo real. Com **Emitir ao iniciar**, o estado atual sai assim que a conta carrega.

| Campo | Conteúdo |
|---|---|
| `msg.payload` | o valor (`true`/`false`, número ou texto) |
| `msg.topic` | o código (ex.: `switch_1`) |
| `msg.device` | id do aparelho |
| `msg.name` | nome do aparelho |
| `msg.status` | todos os códigos e valores atuais |

### Tipo: Leitura (entrada e saída)

Qualquer mensagem na entrada consulta o aparelho e devolve na saída o valor atual do código escolhido (ou de todos), mantendo as propriedades da mensagem de entrada. Com o aparelho offline, dá erro e nada sai.

### Tipo: Comando (entrada; saída opcional)

Envia o comando ao aparelho. Por padrão o nó não tem saída.

**Qual valor enviar**

- **Campo Comando (valor fixo):** tem um seletor de tipo (texto, número ou booleano). Preenchido, qualquer mensagem na entrada envia esse valor ao Código escolhido, sem precisar de *inject* com payload. Com o Código em "Todos", o código vem de `msg.topic`. Para ligar um relé: Código `switch_1`, Comando booleano `true`, e um *inject* qualquer na entrada.
- **Campo Comando vazio:** o valor vem da mensagem:
  - Com um Código escolhido, `msg.payload` é o valor. `ON`, `true` e `1` ligam; `OFF`, `false` e `0` desligam.
  - Com o Código em "Todos", o código vem de `msg.topic` e o valor de `msg.payload`, ou `msg.payload` é um objeto como `{"switch_1": true}`.
- **Comando bruto:** `msg.zeroconf = {comando: "pulse", dados: {pulse: "on", pulseWidth: 1000}}` envia qualquer comando do protocolo local e vale mais que o valor fixo.

**Saída e atraso (para encadear nós)**

- **Ter saída:** marque para o nó ganhar uma saída.
- **Atraso (s):** depois de enviar o comando, o nó espera esse tempo, em segundos.
- **Passar adiante:** texto ou número que vira o `msg.payload` da mensagem emitida na saída. Vazio, a mensagem original segue sem mudança.

Se o comando falhar, nada sai. Se o nó for reimplantado durante a espera, a espera é cancelada. A etiqueta do nó mostra `código=valor` (verde quando verdadeiro), `offline`, `sem chave` (aparelho cifrado sem chave) ou o erro em vermelho.

**Ligar, esperar e seguir.** Tipo: Comando, Código `switch_1`, Comando booleano `true`, **Ter saída** marcada, Atraso `5`, Passar adiante número `1`. Um *inject* liga o relé, o nó espera 5 segundos e emite `payload` = `1`, que você liga ao próximo nó (por exemplo outro Sonoff em Comando, para desligar depois).

### Códigos

| Campo do aparelho | Código no nó |
|---|---|
| `switch` (`on`/`off`) | `switch` (verdadeiro/falso) |
| `switches` (vários canais) | `switch_0`, `switch_1`, ... |
| qualquer outro (`power`, `voltage`, `current`, `rssi`...) | o mesmo nome, valor como veio |

## Rede local dentro de container

Os modos Automático e Local descobrem os aparelhos por mDNS (multicast). Se o Node-RED roda em container Docker, o multicast só chega com rede do tipo **host** (ou macvlan). Em rede bridge o mDNS não funciona; use o modo Nuvem.

## Problemas comuns

| Sintoma | O que fazer |
|---|---|
| Todos os aparelhos aparecem offline | o Node-RED está em outra rede que os aparelhos: use o modo Nuvem |
| Lista de aparelhos vazia | conferir e-mail, senha e país; clicar em **Atualizar lista** |
| Em modo Nuvem a sessão cai (erro 406) | a mesma conta está logada em outro lugar com o mesmo acesso; clicar em **Atualizar lista** |
| Editor mostra o nó como era antes da atualização | reiniciar o Node-RED |

## Atualizar

Baixe o `.tgz` atual e instale do mesmo jeito (Gerenciar paleta, Instalar, enviar o arquivo). Se a atualização mudar os campos do nó, reinicie o Node-RED.

## Desinstalar

Antes de remover o pacote, apague tudo que o usa:

1. Apague **todos os nós Sonoff** de todos os fluxos.
2. Apague a **conta Sonoff salva**: menu, **Configuration nodes**, escolha a conta e exclua.
3. Faça o **Deploy**.
4. Menu, **Gerenciar paleta**, aba **Nós**, ache `sonoff-neway-automacoes` e clique em **Remover**.

## Segurança

A senha fica nas credenciais criptografadas do Node-RED. A lista de aparelhos, com as chaves de cada um, fica em um arquivo `sonoff_dispositivos_<id>.json` na pasta de dados. Quem tem acesso à sua pasta de dados ou ao editor do Node-RED consegue controlar seus aparelhos. Proteja o administrador do Node-RED com senha se ele estiver exposto na rede.

## Limites conhecidos

- Os comandos pela nuvem usam a API REST da eWeLink.
- Medidores de potência, RF Bridge e cortinas mostram o estado como o aparelho envia, sem tratamento próprio.
- Valores como `power` e `voltage` saem sem conversão de escala.
