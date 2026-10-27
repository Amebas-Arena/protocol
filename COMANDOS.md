# Comandos de chat: o contrato

Os SDKs (TypeScript, Python e Go) e o script do painel trazem um sistema de comandos de chat embutido: o jogador digita
`!gols 5`, e o sistema confere permissão, converte os argumentos, controla o cooldown e responde sozinho quando algo
está errado. Este documento é o contrato, pra quem implementa sem SDK. Os casos de `commands.conformance.json` dizem,
mensagem por mensagem, o que tem que acontecer: se a sua implementação passa neles, ela é compatível.

## A define

Cada comando tem um nome e uma **define**: um objeto simples (uma `const`, um `dict`, uma `struct`) que dá pra guardar e
reaproveitar. Tudo é opcional.

| Campo | O que faz |
|---|---|
| `aliases` | Outros nomes do comando (`!g` pra `!gols`). |
| `description` | A linha do `!ajuda` gerado. |
| `hidden` | Tira o comando do `!ajuda`. |
| `args` | Os argumentos, em ordem (veja abaixo). |
| `cooldown` | Espera entre dois usos, em segundos. Um número (por pessoa) ou `{ player, room, exemptStaff }`. |
| `permission` | `all` (padrão), `staff` ou `owner`. |
| `messages` | Troca as respostas prontas: `usage`, `cooldown`, `permission`. |
| `parse` | Troca o parser padrão (veja "Parser próprio"). |

**Padrões globais**: `defaults({ cooldown, permission, messages, prefix, help })` vale pra todos os comandos; a define
de cada um sobrescreve campo a campo. O `prefix` padrão é `!` (de 1 a 3 caracteres, sem espaço).

## Argumentos

Forma curta: `tipo[:faixa][?|=padrão]`.

| Tipo | Aceita | Exemplo | Vira |
|---|---|---|---|
| `int` | inteiro, com `+`/`-`; faixa `1..20`, `1..`, `..20` | `int:1..20` | número inteiro |
| `number` | decimal com `.` ou `,` | `number:0..1` | número |
| `text` | uma palavra; faixa é o tamanho (`text:3..16`) | `text:2..5` | texto |
| `rest` | o resto da linha, com os espaços do meio; tem que ser o último | `rest` | texto |
| `bool` | `sim s yes y true 1 on liga` / `não nao n no false 0 off desliga` | `bool` | verdadeiro/falso |
| `team` | `a` ou `b` | `team` | `"a"` ou `"b"` |
| `player` | `#id`, `@nick`, nick inteiro ou começo do nick (se só um combinar) | `player` | `{ id, nick, accountId }` |
| opções | uma das palavras, sem diferença de maiúscula | `azul\|vermelho` | a opção, como escrita na define |

- `?` deixa o argumento opcional: se faltar, vem `null`. Opcionais só no fim faz sentido, mas não é exigido.
- `=5` dá o valor padrão (e deixa opcional): `int:1..=5`, `int:2..100=6`.
- A forma por extenso é um objeto: `{ "type": "int", "min": 1, "max": 100, "default": 10 }`, ou
  `{ "type": "choice", "choices": ["a", "b"] }`, com `optional`.
- Palavras além das que a define pede são ignoradas (`!gols 7 sobra` roda com `n = 7`).
- Erro de **quem escreve a define** (tipo que não existe, faixa torta, `rest` fora do fim) é lançado na hora de
  registrar, com o que está errado e o jeito certo.

## Respostas prontas

Quando algo está errado, o sistema responde no chat (uma linha pra quem mandou, no script; no bot, uma mensagem na
sala) e **não roda o handler**. Os textos padrão, que a define pode trocar em `messages`:

| Caso | Texto padrão |
|---|---|
| Argumento errado | `{reason}. Uso: {usage}` (ex.: `Valor inválido pra n: um número inteiro de 1 a 20. Uso: !gols <n>`) |
| Faltou argumento | `Faltou n. Uso: !gols <n>` |
| Cooldown | `Espere {left} pra usar {command} de novo.` (`9s`, `1min`, `1min 5s`) |
| Sem permissão | `Só {who} pode usar {command}.` (`a staff` ou `o dono da sala`) |

Variáveis: `{command}` (`!gols`), `{nick}`, `{usage}` (`!gols <n> [modo]`), e por caso `{reason}`, `{left}`, `{who}`.
O uso mostra `<obrigatório>` e `[opcional]`; o `rest` leva reticências (`<motivo...>`).

## Cooldown

- **Por pessoa**: segue a **conta** (sair e voltar com outro id não zera); visitante, pelo nick.
- **Da sala** (`room`): vale pra qualquer pessoa. Os dois se somam: o que acabar depois segura.
- **Staff e dono não esperam**, a não ser que `exemptStaff` seja `false`.
- O cooldown começa **quando o comando passa** (dois `!comando` seguidos não passam juntos) e **volta atrás** se o handler
  devolver `false` (não "usou") ou lançar erro. Argumento errado e falta de permissão nunca gastam cooldown.
- O tempo restante é arredondado pra cima: nunca aparece `0s`.

## O !ajuda gerado

Ligado por padrão (`defaults({ help: false })` desliga). `!ajuda`, `!help` e `!comandos` listam o que **a pessoa pode
usar** (sem os `hidden` e sem os que a permissão dela não alcança), em mensagens de até 200 caracteres.
`!ajuda gols` mostra o uso, a descrição, os atalhos e a espera. Um comando seu chamado `ajuda` (ou `help`) tem
prioridade sobre o embutido.

## Parser próprio

`parse` troca o parser padrão: recebe o texto depois do nome (`rest`, `words`), quem mandou e quem está na sala, e devolve
os args. Pra recusar, lance o erro de comando (`CommandError` / `UsageError`) com o motivo, que vira a resposta de uso.
Nesse caso o handler recebe o que o `parse` devolveu, sem tipagem automática.

## Quem manda e o que o handler recebe

- **Papel** (`player`, `staff`, `owner`): no bot, vem no evento `chat.message` (campo `role`: `owner` é dono da
  comunidade, `staff` é gerente ou moderador).
- O handler recebe o jogador, os args já convertidos, o nome registrado e o digitado (`command`, `alias`), as palavras e
  o resto, e uma função de resposta.
- Devolver `false` (ou o "não contou" da linguagem) não gasta o cooldown.
