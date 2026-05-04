🇧🇷 Português · [🇺🇸 English](./README.en.md)

# skail

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Docs](https://img.shields.io/badge/docs-skail.gitbook.io-blue)](https://skail.gitbook.io/skail-docs/)
[![Site](https://img.shields.io/badge/site-skailhq.com-black)](https://skailhq.com)

**Plataforma de execução durável.** Tolerância a falhas sem PhD em sistemas distribuídos.

Escreva código como se falhas não existissem. Cada `await` vira um checkpoint durável — crash, retry, deploy, reboot — a execução retoma exatamente de onde parou.

```csharp
[SkailFunction]
public async SkailTask ProcessarPedidoAsync(Guid pedidoId)
{
    try
    {
        await ReservarEstoqueAsync(pedidoId);
        var pagamento = await CobrarPagamentoAsync(pedidoId);

        if (pagamento.Aprovado)
        {
            await EmitirNotaFiscalAsync(pedidoId);
            await NotificarPedidoConcluidoAsync(pedidoId);
        }
    }
    catch (Exception ex)
    {
        // Saga: desfaz os passos anteriores
        await EstornarPagamentoAsync(pedidoId);
        await LiberarEstoqueAsync(pedidoId);
        await NotificarPedidoComFalhaAsync(pedidoId, ex.Message);
    }
}
```

Saga durável em `try/catch` nativo. Sem classe `Saga`, sem state machine, sem framework dedicado. Se o servidor cair entre `CobrarPagamentoAsync` e `EmitirNotaFiscalAsync`, a execução retoma em `EmitirNotaFiscalAsync` quando o servidor voltar.

## O que muda na prática

Sem skail, uma saga dessas precisa de:

```
MassTransit/NServiceBus saga         → classe de state machine
Polly retry policies                 → config por chamada externa
Hangfire/Quartz                      → timer pra agendar retentativa
Dead letter queue                    → tratamento de falha irrecuperável
Datadog/ELK                          → observabilidade customizada
──────────────────────────────────────
~400 linhas de infraestrutura por fluxo
```

Com skail: `try/catch`. 35 linhas. O atributo `[SkailFunction]` entrega o resto.

## Agentes IA duráveis

Cada tool call de um agente IA é um checkpoint. Crash no meio de uma cadeia de 47 tool calls? O skail retoma da tool call 48 — sem re-executar as 47 anteriores.

```csharp
[SkailFunction]
public async Task<string> RunAgentAsync(string prompt)
{
    var ctx = new AgentContext(prompt);

    // cada tool call vira um checkpoint durável
    while (!ctx.IsDone)
    {
        var action = await Llm.DecideAsync(ctx);
        var result = await Tool.RunAsync(action);

        // crash aqui? retoma do último checkpoint
        ctx = ctx.With(action, result);
    }

    return ctx.FinalAnswer;
}
```

## O que está embutido

**[Execução durável](https://skail.gitbook.io/skail-docs/)** — cada step é checkpoint. Crash, deploy, reboot: retoma de onde parou. Sem banco de estado, sem fila, sem framework.

**[Retry automático](https://skail.gitbook.io/skail-docs/)** — SefaZ fora do ar? O skail retoma quando ela voltar. Sem `Polly`, sem dead letter queue. Basta escrever `await notaFiscalSrv.EmitirAsync(pedido)` — a resiliência é nativa.

**[Time Travel Debug](https://skail.gitbook.io/skail-docs/)** — reproduza qualquer execução com o estado exato de produção. Rode só o passo que falhou, com os dados reais daquele momento. Sem tentar reproduzir o bug num ambiente diferente.

**[Monitor](https://skail.gitbook.io/skail-docs/)** — cada função vira uma árvore de execução. Cada passo, cada retry, input e output — tudo na timeline. Sem instrumentar código, sem Datadog custom.

**[Zero mudança de paradigma](https://skail.gitbook.io/skail-docs/)** — `[SkailFunction]` num método, `[SkailCommand]` na classe. Seu C# continua C#. Sem DSL, sem JSON de configuração, sem linguagem nova.

## Quick start

```bash
dotnet new skail-app -n MeuProjeto
cd MeuProjeto && dotnet run
```

```bash
curl -X POST "${SKAIL_BASE_URL}/api/v1/trigger/meu-projeto/v1/ProcessarPedido/$(uuidgen)" \
  -H "Authorization: Bearer ${SKAIL_KEY}" -H "Skail-namespace: ${SKAIL_NAMESPACE}" \
  -d '["pedido-001"]'
```

Acompanhe no [Monitor](https://app.skailhq.com): cada passo, cada retry, input e output na timeline.

## Exemplos

| Padrão | O que mostra |
|--------|-------------|
| [`saga-compensation`](./examples/saga-compensation/) | Saga com compensação automática via `try/catch` nativo |
| [`billing-retry`](./examples/billing-retry/) | Retry automático em falha externa + `Delay` de 24h sem cron |
| [`webhook-bridge`](./examples/webhook-bridge/) | Hibernação durável esperando sinal externo via `WaitForEvent` |
| [`human-approval`](./examples/human-approval/) | Aprovação humana com prazo: `WhenAny(WaitForEvent, Delay)` |
| [`polling-timeout`](./examples/polling-timeout/) | Operação externa lenta vs. deadline — `WhenAny` por referência |
| [`hello-world`](./examples/hello-world/) | A menor função durável possível |

Cada exemplo compila e roda. Copie, cole, `dotnet run`.

## Linguagens

**C# (.NET 8+) hoje · qualquer linguagem via REST.** Dois endpoints HTTP, três cabeçalhos — Node, Python, Go, Java, Delphi, Ruby, qualquer sistema que faz `POST`. Detalhes em [`llms.txt`](./llms.txt).

## Mais

- [Docs](https://skail.gitbook.io/skail-docs/) — getting started, padrões, referência de API
- [`llms.txt`](./llms.txt) — referência condensada para Claude, Cursor, Copilot

---

Em produção: **Nota Gateway** (milhões de transações fiscais/mês).

*freedom to scale.*
