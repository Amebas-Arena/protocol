# Protocolo do bot de sala do Amebas Arena

O contrato do WebSocket `/v1/rooms/{roomId}/bot`, em JSON Schema: a **fonte única** de onde os SDKs de todas as
linguagens geram os tipos. Quem não usa SDK lê este arquivo e fala o protocolo direto (exemplo em JavaScript
puro abaixo).

| Arquivo | O que é |
|---|---|
| `bot-protocol.schema.json` | O schema (JSON Schema 2020-12): `ClientMessage` (o que o bot manda) e `ServerMessage` (o que a API manda), mais as regras do canal em `x-amebas-bot` (rota, prazo da auth, limite de comandos, códigos de fechamento, lista de comandos e eventos). |
| `commands.conformance.json` | Os casos de teste do sistema de comandos de chat (`!gols 5`). Todo SDK passa nos mesmos. |
| [`COMANDOS.md`](COMANDOS.md) | O contrato do sistema de comandos de chat, pra quem implementa sozinho. |

O arquivo **não é escrito à mão**: sai do código do servidor (os mesmos schemas Zod que a API usa pra validar o canal), e
um teste do jogo falha se ele ficar diferente do servidor. Mudou o protocolo? Gera-se de novo, sobe a versão
(tag) e os SDKs acompanham.

## Os SDKs

| Linguagem | Pacote | Repositório |
|---|---|---|
| TypeScript / JavaScript | `npm install @amebas/sdk` | [Amebas-Arena/sdk-js](https://github.com/Amebas-Arena/sdk-js) |
| Python | `pip install amebas` | [Amebas-Arena/sdk-python](https://github.com/Amebas-Arena/sdk-python) |
| Go | `go get github.com/Amebas-Arena/sdk-go` | [Amebas-Arena/sdk-go](https://github.com/Amebas-Arena/sdk-go) |

### Quero um SDK pra outra linguagem

Pegue o `bot-protocol.schema.json` numa tag fixa e gere os tipos com o gerador de JSON Schema da linguagem:

```sh
# o arquivo de uma versão (tag) específica
curl -LO https://raw.githubusercontent.com/Amebas-Arena/protocol/v1.0.0/bot-protocol.schema.json
```

| Linguagem | Gerador | Exemplo |
|---|---|---|
| Rust | [typify](https://github.com/oxidecomputer/typify) | `cargo typify bot-protocol.schema.json` |
| Java / Kotlin | [jsonschema2pojo](https://www.jsonschema2pojo.org) | `jsonschema2pojo --source bot-protocol.schema.json --target src/main/java` |
| C# | [quicktype](https://quicktype.io) | `quicktype -s schema bot-protocol.schema.json -o Protocol.cs` |
| PHP, Swift, Dart, Ruby... | [quicktype](https://quicktype.io) | `quicktype -s schema bot-protocol.schema.json -o protocolo.swift` |

Todo comando e todo evento tem um tipo com nome próprio (`ChatCommand`, `PlayerJoinedEvent`...), que é o que os
geradores precisam. O resto do SDK é pequeno: abrir o socket, mandar a auth, casar `ref` com `result` e reconectar
(veja `x-amebas-bot` no schema). Depois, o sistema de comandos de chat: [`COMANDOS.md`](COMANDOS.md) e os casos de
`commands.conformance.json`.

## Sem SDK: o protocolo em 25 linhas de JavaScript

```js
// Node 22+ ou navegador. A chave é uma chave de API da comunidade com o escopo rooms:control (amb_...).
const ws = new WebSocket('wss://SEU-SITE/api/v1/rooms/ID_DA_SALA/bot');
let ref = 0;
const pending = new Map();

const send = (command) =>
  new Promise((resolve, reject) => {
    const id = `c${++ref}`;
    pending.set(id, { resolve, reject });
    ws.send(JSON.stringify({ ...command, ref: id })); // o ref volta no result
  });

ws.onopen = () => ws.send(JSON.stringify({ type: 'auth', key: 'amb_...' })); // sempre a primeira mensagem
ws.onmessage = ({ data }) => {
  const m = JSON.parse(data);
  if (m.type === 'ready') return send({ type: 'chat', text: 'Bot no ar!' });  // auth aceita
  if (m.type === 'result') {                                                   // resposta de um comando
    const p = pending.get(m.ref);
    pending.delete(m.ref);
    return m.ok ? p?.resolve() : p?.reject(new Error(`${m.code}: ${m.message}`));
  }
  if (m.type === 'error') return console.error(m.code, m.message);            // auth recusada etc.
  if (m.type === 'player.joined') send({ type: 'chat', text: `Oi, ${m.player.nick}!` }); // um evento da sala
};
```

O que o servidor garante:

- **Auth**: a primeira mensagem é `{ "type": "auth", "key": "amb_..." }` (a chave nunca vai na URL). Quem fala HTTP
  com cabeçalho pode mandar `Authorization: Bearer amb_...` no upgrade. Sem auth em 10 s, o socket fecha com `4001`.
- **Erros de auth**: vêm como `{ "type": "error", "code": "unauthorized" | "forbidden" | "not_found", ... }` seguido do
  fechamento `4001`. Não adianta tentar de novo com a mesma chave. Sala inexistente ou id inválido fecha com `4004`.
- **Comandos**: qualquer comando da sala (`chat`, `kick`, `ban`, `start`...) mais um `ref` opcional (texto de até 64
  caracteres). O `result` volta com o mesmo `ref`. Limite: 20 de cara e 5 por segundo; passou, o `result` vem com
  `ok: false` e `code: "rate_limited"`.
- **Eventos**: chegam sozinhos, sem pedir. A lista está em `x-amebas-bot.events`.
- **Cair é normal**: reconecte com espera crescente e mande a auth de novo. Os `ref` pendentes se perdem.
