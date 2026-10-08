# demo-support-agent

Agente **A2A** de suporte de pedidos da TechWave Electronics (demo do MuleSoft Meetup).
Recebe uma pergunta em linguagem natural, decide quais tools MCP chamar via tool-calling
(Azure OpenAI `/responses`) e devolve a resposta como uma task A2A.

- **Protocolo:** A2A 1.0.0 (A2A Connector 2.0) · JSON-RPC em `${config.agent.path}/rpc`
  (`/techwave-order-support-agent/rpc`) — o `agentPath` tem que ser igual ao base path
  público da instância no gateway, ver a nota de bijeção abaixo
- **Consome:** [`demo-support-mcp-server`](../demo-support-mcp-server) via MCP (Streamable HTTP)
- **Runtime:** Mule 4.12 · Java 17 · CloudHub 2.0

> Parte de uma demo com quatro repositórios, que só faz sentido completa:
>
> - [`demo-order-support-api`](https://github.com/mule-meetups-ctba/demo-order-support-api) — API REST de pedidos + consulta do pagamento no Stripe
> - [`demo-support-mcp-server`](https://github.com/mule-meetups-ctba/demo-support-mcp-server) — MCP server com as quatro tools de suporte
> - [`demo-omni-agent-network`](https://github.com/mule-meetups-ctba/demo-omni-agent-network) — Agent Network 2.0 + broker em AgentScript
>
> A arquitetura, o passo a passo de deploy e as políticas de gateway estão
> descritos no artigo que acompanha a demo.

## Como funciona

```
A2A task-listener  →  LLM (escolhe tools)  →  MCP call-tool xN  →  LLM (resposta final)  →  A2A task
```

Dois pontos de desenho que valem a leitura do código:

- **`<a2a:interfaces>`** em `src/main/mule/global-config.xml` declara o binding JSON-RPC no path
  `/rpc`. O connector valida no startup uma **bijeção** entre cada `<a2a:interface>` e as entradas
  de `supportedInterfaces` do agent card — mexeu num lado, mexa no outro, senão o app não sobe.
- **`<mcp:default-request-headers>`** carrega `client_id`/`client_secret` do contrato
  `agent-to-mcp-server`. Sem eles, o gateway devolve `401` antes de a chamada chegar ao MCP server.
  Atenção ao XSD: `<reconnection>` é filho de `<mcp:client-config>`, **não** da connection.

## Configuração

`src/main/resources/config-dev.yaml`:

| Propriedade | O que é |
|---|---|
| `config.agent.publicUrl` | URL pública do agente via ingress Omni Gateway |
| `config.openai.host` / `.basePath` / `.model` | endpoint e deployment do Azure OpenAI |
| `config.mcp.serverUrl` / `.endpointPath` | onde encontrar o MCP server (rota governada) |
| `config.mcp.clientId` | `client_id` do contrato `agent-to-mcp-server` |

Segredos (cifrados em `secure-config-dev.yaml`): `openai.api.key`, `mcp.clientSecret`.
Ver [SECRETS.md](SECRETS.md).

## Build e deploy

```bash
# testes
mvn clean test

# 1) publicar o asset no Exchange
mvn clean deploy -DskipTests

# 2) deploy no CloudHub 2.0
mvn mule:deploy -DmuleDeploy -DskipTests \
  -Dconnected.app.client.id=<CLIENT_ID> \
  -Dconnected.app.client.secret=<CLIENT_SECRET> \
  -Danypoint.org.id=<ORG_ID> \
  -Ddeploy.target=<space ou região> \
  -Denvironment=Sandbox \
  -Dencryption.key=<CHAVE_16_CHARS>
```

Pré-requisitos: JDK 17, Maven 3.9.x e um `<server id="anypoint-exchange-v3">` no
`~/.m2/settings.xml` com as credenciais do Connected App.

## Testando

```bash
# agent card
curl https://<ingress-gw>/techwave-order-support-agent/.well-known/agent-card.json | jq

# SendMessage (A2A 1.0: PascalCase, ROLE_USER, Part sem campo "kind")
curl -X POST https://<ingress-gw>/techwave-order-support-agent/rpc \
  -H "Content-Type: application/json" \
  -H "client_id: ..." -H "client_secret: ..." \
  -d '{"jsonrpc":"2.0","id":"1","method":"SendMessage","params":{"message":{
      "messageId":"11111111-1111-1111-1111-111111111111","role":"ROLE_USER",
      "parts":[{"text":"Qual o status do pedido TW-1001?"}]}}}' | jq
```

Esperado: `result.status.state = "TASK_STATE_COMPLETED"`.
