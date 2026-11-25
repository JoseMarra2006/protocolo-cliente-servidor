# Casos de conformidade — Reservas de pedido (3ª entrega)

Este documento lista casos de teste para conferir se um **servidor** ou um **cliente** segue a versão 1.6.0 do protocolo nas reservas de pedido (`/orders`, regras 18 a 23 do [README](./README.md)) e se a 1ª e a 2ª entregas continuam funcionando como antes. Cada caso diz a requisição, o resultado esperado e a regra de onde ele vem. Dá para executar com qualquer cliente HTTP (Postman, Insomnia, `curl`) ou com um script na linguagem que você preferir. Para testar um servidor, envie as requisições e compare as respostas. Para testar um cliente, rode-o contra um servidor que passa nestes casos e confira se ele mostra cada situação corretamente.

Todas as rotas começam com `/api/v1` (regra 1), omitido nas tabelas. Corpos são JSON com `Content-Type: application/json` (regra 2), e as rotas protegidas levam `Authorization: Bearer <token>` (regra 3). Os ids usados nas rotas vêm sempre do corpo das respostas, nunca do `Location` (regra 17).

## Preparação

1. **ADM** — `POST /sessions` com `{"email": "adm@teste.com", "senha": "admin123"}`. Guardar o `token` (token do ADM) e o `id` da sessão. O ADM padrão existe em todo servidor (seção "ADM padrão" do README).
2. **Produto P** — com o token do ADM, `POST /products` com `{"nome": "Produto P", "descricao": "Produto de teste", "quantidade": 10}`. Guardar o `id` (id de P).
3. **Usuários A e B** — `POST /users` com `{"nome": "Usuario A", "email": "usuario.a@teste.com", "senha": "senha123"}` e `{"nome": "Usuario B", "email": "usuario.b@teste.com", "senha": "senha123"}`. Se o e-mail já existir no servidor (`409`), use outro.
4. **Login de A e de B** — `POST /sessions` com o e-mail e a senha de cada um. Guardar o token de A e o token de B.

Convenções usadas nas tabelas:

- **Estoque de P** é o campo `quantidade` de `GET /products/{id de P}` (regra 19).
- **Estado inicial de cada grupo:** estoque de P igual a 10 e nenhuma reserva ativa de A, de B ou do ADM. Para chegar a esse estado, cancele as reservas que sobraram (`GET /orders` e `DELETE /orders/{id}` com o token de cada usuário) e, depois, o ADM faz `PATCH /products/{id de P}` com `{"quantidade": 10}` (regra 19).
- Dentro de um grupo, os casos rodam na ordem da tabela e cada um parte do estado deixado pelo anterior, salvo quando o caso diz outro estoque inicial; nesse caso o ADM ajusta o estoque com `PATCH /products/{id de P}` antes do caso.
- `{R}` é o `id` de uma reserva criada no próprio caso ou no caso anterior.
- Toda resposta de erro deve ter o corpo `{"mensagem": "texto"}` (regra 6); as tabelas indicam só o código.

Observação: o documento não tem casos com quantidades como `1.0` ou `1e3`, porque a regra 13 deixa essa forma a critério do servidor.

## Criar (`POST /orders`)

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| C01 | A envia `POST /orders` com `{"produtoId": <id de P>, "quantidade": 2}`. | `201`, cabeçalho `Location` igual a `/api/v1/orders/{id}` com o `id` do corpo. Corpo `OrderResponse` com `produto.id` igual ao id de P, `quantidade` 2, `criadaEm` e `expiraEm` no formato `AAAA-MM-DDTHH:MM:SSZ`, `expiraEm` igual a `criadaEm` mais 180 segundos e `tempoRestanteSegundos` 180 (aceitar 179 se a resposta cruzar a virada de um segundo). | 17, 18, 20 |
| C02 | Em seguida, `GET /products/{id de P}`. | `200` e estoque de P igual a 8. | 19 |
| C03 | O ADM envia `POST /orders` com `{"produtoId": <id de P>, "quantidade": 1}`. | `201`. | 21 |
| C04 | Estoque inicial de P: 5. A reserva exatamente 5 unidades de P. | `201` e estoque de P igual a 0. | 19, 22 |
| C05 | Estoque inicial de P: 3. A pede 4 unidades de P. | `409` e estoque de P continua 3. | 19, 22 |
| C06 | Estoque inicial de P: 0. A pede 1 unidade de P. | `409`. | 22 |
| C07 | A envia `{"produtoId": <id que não existe>, "quantidade": 1}`. | `404`. | 22 |
| C08 | A envia, um de cada vez, `produtoId` igual a `0`, `-1`, `1.5`, `"7"` e `null`, e também um corpo sem `produtoId` (sempre com `"quantidade": 1`). | `400` em todos. | 13, 22 |
| C09 | A envia, um de cada vez, `quantidade` igual a `0`, `-1`, `1.5`, `"2"`, `null` e `1000001`, e também um corpo sem `quantidade` (sempre com o id de P). | `400` em todos. | 13, 22 |
| C10 | A envia um corpo JSON mal formado, por exemplo `{"produtoId": 1,`. | `400`. | 22 |
| C11 | `POST /orders` com corpo válido, primeiro sem o cabeçalho `Authorization` e depois com um token inventado. | `401` nos dois. | 3, 22 |
| C12 | A envia `{"produtoId": <id que não existe>, "quantidade": 0}`. | `400` (o corpo é validado antes da existência do produto). | 22 |
| C13 | A envia `{"produtoId": <id de P>, "quantidade": 1, "observacao": "teste"}`. | `201`; o campo desconhecido é ignorado. | 7 |
| C14 | A envia duas vezes `{"produtoId": <id de P>, "quantidade": 1}`. | Duas respostas `201`, com `id` diferentes. | 18 |

## Consultar e listar (`GET /orders/{id}` e `GET /orders`)

Antes do grupo: A cria a reserva `{R}` com `{"produtoId": <id de P>, "quantidade": 2}` e guarda o corpo da resposta.

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| R01 | A envia `GET /orders/{R}`. | `200` com o mesmo `id`, `produto`, `quantidade`, `criadaEm` e `expiraEm` da criação, e `tempoRestanteSegundos` menor ou igual ao da criação. | 20, 21 |
| R02 | B envia `GET /orders/{R}`. | `403`. | 21 |
| R03 | O ADM envia `GET /orders/{R}`. | `403`. | 10, 21 |
| R04 | A e o ADM enviam `GET /orders/{id que não existe}`. | `403` para os dois. | 10, 21 |
| R05 | A envia `GET /orders/abc` e `GET /orders/1.5`. | `400` nos dois (o id mal formatado é conferido antes da permissão). | 22 |
| R06 | `GET /orders/{R}` sem o cabeçalho `Authorization`. | `401`. | 3, 22 |
| R07 | A cria mais uma reserva de 1 unidade de P. Depois: A envia `GET /orders`; B (sem reservas) envia `GET /orders`; o ADM envia `GET /orders`. | A: `200` com um array só com as reservas ativas de A, em ordem de `id` crescente. B: `200` com `[]`. ADM: `200`, sem nenhuma reserva de A. | 16, 21 |
| R08 | O ADM envia `PATCH /products/{id de P}` com `{"nome": "Produto P renomeado"}`; depois A envia `GET /orders/{R}`. | `200` e `produto.nome` igual a `Produto P renomeado`. | 23 |

## Atualizar (`PUT` e `PATCH` em `/orders/{id}`, mesmo efeito)

Antes do grupo: A cria a reserva `{R}` com `{"produtoId": <id de P>, "quantidade": 2}` (estoque de P passa a 8) e guarda `criadaEm` e `expiraEm`.

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| U01 | A envia `PUT /orders/{R}` com `{"quantidade": 3}`. | `200`, `quantidade` 3, `criadaEm` e `expiraEm` iguais aos da criação. Estoque de P diminui 1 (vai a 7). | 19, 20 |
| U02 | A envia `PATCH /orders/{R}` com `{"quantidade": 2}` e, em seguida, `PATCH /orders/{R}` com `{"quantidade": 3}`. | `200` nos dois, com o mesmo efeito do `PUT`: estoque de P vai a 8 e depois a 7. | 19 |
| U03 | A envia `PUT /orders/{R}` com `{"quantidade": 1}`. | `200`, `quantidade` 1. Estoque de P aumenta 2 (vai a 9). | 19 |
| U04 | A envia `PUT /orders/{R}` com `{"quantidade": 1}` de novo. | `200`, sem mudança; estoque de P continua 9. | 19 |
| U05 | A envia `PATCH /orders/{R}` com `{"quantidade": 11}` (precisaria de 10 a mais, e o estoque é 9). | `409`. A reserva continua com `quantidade` 1 e o estoque de P continua 9. | 19, 22 |
| U06 | A envia `PUT` e `PATCH /orders/{R}` com `quantidade` igual a `0`, `"2"`, `1.5` e `null`, e com um corpo sem `quantidade` (`{}`). | `400` em todos. | 13, 22 |
| U07 | O ADM cadastra um produto Q. A envia `PUT /orders/{R}` com `{"produtoId": <id de Q>, "quantidade": 2}`. | `200`, `quantidade` 2 e `produto.id` continua o id de P; o `produtoId` é ignorado. | 7 |
| U08 | B envia `PUT /orders/{R}` com `{"quantidade": 0}`. | `403` (a permissão é conferida antes do corpo). | 21, 22 |
| U09 | A envia `PUT /orders/{id que não existe}` com `{"quantidade": 1}`. | `403`. | 21 |
| U10 | A envia `GET /orders/{R}`. | `criadaEm` e `expiraEm` iguais aos da criação; nenhuma atualização renovou o prazo. | 20 |

## Cancelar (`DELETE /orders/{id}`)

Antes do grupo: A cria a reserva `{R}` com `{"produtoId": <id de P>, "quantidade": 2}` (estoque de P passa a 8).

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| D01 | A envia `DELETE /orders/{R}`; logo em seguida, `GET /products/{id de P}`. | `204` sem corpo. Estoque de P de volta a 10. | 19 |
| D02 | A envia `DELETE /orders/{R}` de novo. | `403`. | 21 |
| D03 | A envia `GET /orders/{R}` e `GET /orders`. | `403` na consulta; a lista não contém `{R}`. | 18, 21 |
| D04 | A cria uma nova reserva `{R}`. B envia `DELETE /orders/{R}`; depois A envia `GET /orders/{R}`. | B: `403`. A: `200`, reserva intacta. | 21 |
| D05 | A envia `DELETE /orders/abc`; depois `DELETE /orders/{R}` sem o cabeçalho `Authorization`. | `400` no primeiro; `401` no segundo. | 22 |

## Expirar (caso lento: cerca de 3 minutos)

Antes do grupo: A cria a reserva `{R}` com `{"produtoId": <id de P>, "quantidade": 2}` (estoque de P passa a 8) e guarda `expiraEm` e `tempoRestanteSegundos`.

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| X01 | Antes de terminar o `tempoRestanteSegundos` recebido, A envia `GET /orders/{R}` e `GET /orders`. | `200` na consulta, e a lista contém `{R}`. | 20 |
| X02 | Depois de passado o `tempoRestanteSegundos` recebido na criação (aguarde alguns segundos a mais), A envia `GET /orders/{R}`, `PUT /orders/{R}` com `{"quantidade": 1}`, `DELETE /orders/{R}` e `GET /orders`. | `403` na consulta, na atualização e no cancelamento; a lista não contém `{R}`. | 18, 20, 21 |
| X03 | Depois da expiração, `GET /products/{id de P}`. | Estoque de P de volta a 10, o valor anterior à reserva. | 19 |

## Efeitos cruzados

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| E01 | A cria uma reserva de 3 unidades de P (estoque vai a 7). O ADM envia `DELETE /users/{id de A}`; depois, `GET /products/{id de P}` e qualquer requisição protegida com o token de A. | `204` na remoção. Estoque de P de volta a 10 já na primeira leitura após o `204`. O token de A dá `401`. Antes dos próximos casos, cadastre A de novo e faça login. | 3, 23 |
| E02 | O ADM cadastra um produto Q com `quantidade` 5. A reserva 2 unidades de Q (reserva `{R}`). O ADM envia `DELETE /products/{id de Q}`; depois A envia `GET /orders/{R}` e `GET /orders`. | `204` na remoção do produto, mesmo com a reserva ativa. A consulta dá `403` e a lista não contém `{R}`. | 21, 23 |
| E03 | A cria a reserva `{R}`, faz logout (`DELETE /sessions/{id da sessão}`), faz login de novo e, ainda dentro dos 180 segundos, envia `GET /orders`. | A lista contém `{R}`; o logout não cancela reservas. | 21, 23 |
| E04 | Com a reserva `{R}` de A ativa, o ADM envia `PUT /products/{id de P}` (ou `PATCH`) com `quantidade` 50; depois, `GET /products/{id de P}` e A envia `GET /orders/{R}`. | Estoque de P igual a 50. A reserva continua ativa (`200`), com a mesma `quantidade`. | 19 |
| E05 | A faz um segundo login (sessão 2). Com o token da sessão 1 e com o da sessão 2, A envia `GET /orders`. | As duas listas são iguais. | 14, 21 |
| E06 | Com o id do produto Q removido em E02, A envia `POST /orders` com `{"produtoId": <id de Q>, "quantidade": 1}`. | `404`. | 22 |
| E07 | O ADM cadastra um produto S com `quantidade` 10. A reserva 2 unidades de S (estoque de S vai a 8). O ADM define `quantidade` 50 em S (`PUT` ou `PATCH`) e consulta o estoque; depois A cancela a reserva e o ADM consulta o estoque de novo. | Depois do ADM, o estoque de S é 50 e a reserva continua ativa; depois do cancelamento, o estoque de S é 52. | 19 |

## Concorrência

Para disparar requisições em paralelo em qualquer linguagem: crie N tarefas (threads, promises, goroutines ou processos), cada uma com a própria conexão HTTP e já com a requisição montada; faça todas esperarem um mesmo sinal de largada e libere o sinal de uma vez. Guarde o código de status de cada resposta e, depois que todas terminarem, leia o estoque de P. Repita algumas vezes: um servidor sem atomicidade pode passar por acaso numa execução e falhar na seguinte. As requisições podem usar o token de A (não há limite de reservas por usuário, regra 18).

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| K01 | Estoque inicial de P: 1. 20 `POST /orders` simultâneos com `{"produtoId": <id de P>, "quantidade": 1}`. | Exatamente um `201` e dezenove `409`. Estoque de P igual a 0. | 19 |
| K02 | Estoque inicial de P: 10. 30 `POST /orders` simultâneos de 1 unidade de P. | Exatamente dez `201` e vinte `409`. Estoque de P igual a 0. | 19 |
| K03 | A cria a reserva `{R}` de 2 unidades de P (estoque vai a 8). 5 `DELETE /orders/{R}` simultâneos com o token de A. | Exatamente um `204` e quatro `403`. Estoque de P igual a 10: a quantidade foi devolvida uma única vez. | 19, 21 |
| K04 | Estoque inicial de P: 10. Vários `POST /orders` de quantidades variadas e `DELETE /orders/{id}` das reservas criadas, misturados e em paralelo, todos dentro dos 180 segundos; durante o teste, leituras de `GET /products/{id de P}` em paralelo. | Ao final, estoque de P mais a soma das `quantidade` das reservas ativas (`GET /orders`) é igual a 10. Nenhuma leitura mostrou estoque negativo. | 19 |

## CORS e métodos

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| M01 | `OPTIONS /orders` e `OPTIONS /orders/{R}`, sem token e sem corpo. | `204` sem corpo, com `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS` e `Access-Control-Allow-Headers: Content-Type, Authorization`. | 5 |
| M02 | Repetir C11 (`401`), R02 (`403`) e C05 (`409`) olhando os cabeçalhos. | Os três cabeçalhos CORS de M01 em todas as respostas, inclusive nas de erro. | 5 |
| M03 | `DELETE /orders` e `POST /orders/{R}`, com o token de A. | `405` nos dois, com corpo `ErroResponse` e cabeçalhos CORS. | 15 |
| M04 | Conferir o corpo de todas as respostas de erro dos grupos anteriores. | Sempre `{"mensagem": "texto"}`, com `mensagem` texto. | 6 |

## Regressão (1ª e 2ª entregas)

| Caso | Passos | Resultado esperado | Regra |
|---|---|---|---|
| G01 | Login do ADM; `GET /users` com o token do ADM e com o de A; o ADM cadastra, consulta, atualiza com `PUT` e `PATCH` e remove um produto; A envia `DELETE /sessions/{id que não existe}`, e o ADM também. | Login `201`. `GET /users`: `200` para o ADM e `403` para A. CRUD de produto: `201`, `200`, `200`, `200` e `204`, sem nenhum `409`. `DELETE /sessions/{id que não existe}`: `403` para A e para o ADM. | 10, 17 |
| G02 | `GET /products` com o token de A. | `200`, array em ordem de `id` crescente; cada produto com os quatro campos `id`, `nome`, `descricao` e `quantidade`, sem campos de reserva. | 16, 19 |

## Como relatar divergências

Se o seu servidor ou cliente, ou o de um colega, não passar num caso, ou se um caso parecer contradizer o README ou o YAML, abra uma issue neste repositório.
Informe o id do caso (por exemplo, `K01`), a requisição enviada (método, rota, cabeçalhos e corpo) e a resposta obtida (código, cabeçalhos e corpo).
Se a divergência estiver no próprio protocolo, siga a seção "Como contribuir" do README.
