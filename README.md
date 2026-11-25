# Ciba AI

Repositório e infraestrutura de servidor para a assistente de IA Garcia e futuras integrações.

## Objetivos

- Backend independente que fornece inferência e APIs à assistente Garcia; integrações com outros aplicativos serão adicionadas futuramente.
- Acesso privado de proprietário único: a Garcia só pode ter um proprietário ativo e um dispositivo registrado por vez; não há cadastro público ou fluxo multiusuário.
- Acesso controlado por convites de uso único e token de dispositivo protegido.
- Núcleo Ciba Core com roteamento por especialidade e identidade unificada.
- Provider atual Cloudflare Workers AI e adapter preparado para endpoints OpenAI-compatible auto-hospedados.
- Perfis especialistas selecionados sem chamadas adicionais no modo econômico single-pass.
- Memória curada e privada do proprietário, separada do histórico; conversas não são persistidas pelo gateway.
- Primitiva opt-in de ensemble para até três especialistas independentes e uma síntese, ainda desativada sem providers configurados.
- Registro de providers e ferramentas opt-in com allowlist, validação de entrada/saída e autorização pela aplicação.
- Sem fallback pago, cobrança automática ou uso de credenciais Google do mantenedor.
- Limites diários por dispositivo, aplicativo e instalação.
- Não persistir o conteúdo das conversas.

## Custo zero: condição obrigatória

A franquia gratuita documentada pela Cloudflare é de 10.000 Neurons por dia. No Workers Free, as chamadas falham ao atingir o limite; no Workers Paid, o excedente é cobrado a US$ 0,011 por 1.000 Neurons. O modelo Gemma 4 está listado a 9.091 Neurons por milhão de tokens de entrada e 27.273 por milhão de tokens de saída. Os tetos internos foram reduzidos para criar margem dentro da franquia, mas não representam garantia matemática de consumo: a contagem real deve ser acompanhada no painel. [Tabela oficial por modelo](https://developers.cloudflare.com/workers-ai/platform/pricing/). Para manter custo zero, use uma conta Workers Free, não habilite Workers Paid nem créditos pré-pagos do AI Gateway e não configure fallback pago. O limite interno de requisições do Ciba AI é uma proteção adicional, não uma conversão exata de Neurons. Confira o painel de consumo da Cloudflare regularmente. [Documentação oficial de preços](https://developers.cloudflare.com/workers-ai/platform/pricing/).

## Estado do projeto

A assistente se chama **Garcia**. **Ciba AI** é somente o nome deste repositório e da infraestrutura de servidor Cloudflare que fornece a API. A Garcia é o produto de IA com identidade e aplicativo próprios. O Worker Ciba AI está publicado em `https://ciba-ai.fabiociba7.workers.dev`, com D1 conectado, migrations aplicadas, autenticação por dispositivo e limites de uso. O endpoint `/v1/status` confirmou o serviço pronto e a IA habilitada; uma inferência real ainda deverá ser validada na rodada final com o aplicativo. O projeto Android independente da Garcia está em [garcia-android/](garcia-android/), com interface nativa, conversa, histórico local criptografado e conexão com a API. A rota de pareamento do cliente corresponde a `POST /v1/owner/pair`. Após publicar a configuração que permite `com.ciba.garcia`, execute uma única vez o workflow `Provision Garcia Device`, confirmando a operação, para criar o proprietário privado e emitir os códigos únicos de ativação. O workflow `Ciba AI Deploy` publica automaticamente mudanças de backend na branch `main`, aplica migrations e verifica a saúde do Worker. Mudanças exclusivas no app Android ou na documentação não disparam deploy do servidor. O registro de ferramentas externas está conectado ao orquestrador com validação, autorização e aprovação durável; integrações externas continuam opt-in.

## Infraestrutura e expansão

Consulte [INFRASTRUCTURE_AUDIT.md](docs/INFRASTRUCTURE_AUDIT.md) para a auditoria dos limites gratuitos, estado confirmado em produção e critérios de expansão sem custos.

## Arquitetura

A camada de inteligência unificada está descrita em [UNIFIED_INTELLIGENCE.md](docs/UNIFIED_INTELLIGENCE.md), e a seleção de projetos open source em [OPEN_SOURCE_SELECTION.md](docs/OPEN_SOURCE_SELECTION.md). O núcleo atual usa um único modelo por chamada e seleciona um perfil especialista; não afirma executar vários modelos independentes. O adapter para providers OpenAI-compatible está preparado, mas ainda não conectado nem configurado.


```
App Ciba
  |-- POST /v1/auth/register  (convite de uso único)
  |-- POST /v1/ai/chat        (Bearer token do dispositivo)
  |-- GET  /v1/usage
  v
Ciba AI Worker
  |-- autenticação e autorização
  |-- cotas atômicas em D1
  |-- Cloudflare Workers AI
  v
Resposta ao app (sem armazenar o histórico)
```

## Identidade exclusiva da Garcia

A Garcia é uma assistente privada de Ciba, não um serviço multiusuário. O backend permite apenas um dispositivo ativo para `com.ciba.garcia`, impede a emissão de convites adicionais enquanto esse dispositivo estiver ativo e não permite criar outro proprietário para a mesma instalação. Para substituir o aparelho, o dispositivo anterior deve ser revogado pelo administrador antes de um novo provisionamento. A memória curada inicial identifica Ciba como criador, proprietário, mentor, amigo e parceiro da Garcia; nenhum histórico de conversa é importado automaticamente.

## Aplicativo Android Garcia

O aplicativo Android é um projeto independente dentro de `garcia-android/`; não depende do GEIVIA ou do Zupi. Consulte o [README da Garcia](garcia-android/README.md) para arquitetura, abertura no Android Studio e ativação privada de dispositivo único.

## Desenvolvimento

Requisitos: Node.js 20+ e conta Cloudflare.

```bash
npm install
npm run check
npx wrangler d1 create ciba-ai-db
```

Copie o ID retornado para `wrangler.toml`. Confirme os package IDs Android permitidos em `ALLOWED_APP_IDS` (a Garcia usa `com.ciba.garcia`), aplique as migrations e configure os segredos:

```bash
npx wrangler d1 migrations apply ciba-ai-db --remote
npx wrangler secret put ADMIN_TOKEN
npx wrangler deploy
```

O nome do modelo é configurado em `AI_MODEL`. Antes de produção, confirme no catálogo da Cloudflare o identificador e a disponibilidade do modelo escolhido para a sua conta.

## Variáveis

| Variável | Uso |
|---|---|
| `AI_MODEL` | Identificador do modelo Workers AI |
| `ALLOWED_APP_IDS` | IDs de apps Ciba separados por vírgula |
| `DAILY_GLOBAL_LIMIT` | Requisições globais/dia (teto 50) |
| `DAILY_APP_LIMIT` | Requisições por app/dia (teto 25) |
| `DAILY_DEVICE_LIMIT` | Requisições por dispositivo/dia (teto 10) |
| `MAX_OUTPUT_TOKENS` | Limite de saída solicitado ao modelo (teto 512) |
| `ADMIN_TOKEN` | Segredo para operações administrativas |

## Rotas

- `GET /v1/status`
- `GET /v1/config` (configuração pública para os aplicativos)
- `GET /v1/models`
- `GET /v1/ai/capabilities`
- `POST /v1/auth/register`
- `POST /v1/ai/chat`
- `GET /v1/usage`
- `GET /v1/admin/apps`
- `POST /v1/admin/apps`
- `POST /v1/admin/invitations`
- `POST /v1/admin/devices/revoke`
- `GET /v1/admin/ai/overview` (visão protegida para futura tela administrativa)
- `GET /v1/admin/settings`
- `POST /v1/admin/settings`

A integração MCP HTTP da Garcia está descrita em [MCP.md](docs/MCP.md). A biblioteca de skills está documentada em [SKILLS.md](docs/SKILLS.md). A integração Composio está descrita em [COMPOSIO.md](docs/COMPOSIO.md). Consulte [TOOLS.md](docs/TOOLS.md) para ativar ferramentas com autorização explícita e conhecer os limites do registro. Consulte [PROVISIONING.md](docs/PROVISIONING.md) e o workflow `Provision Garcia Device` para o provisionamento privado do dispositivo. Consulte [TESTING.md](docs/TESTING.md) para executar os testes locais sem credenciais Cloudflare. Consulte [APP_INTEGRATION.md](docs/APP_INTEGRATION.md) para o contrato de integração dos clientes. Consulte [SECURITY.md](docs/SECURITY.md) antes de disponibilizar convites ou distribuir os aplicativos. A futura tela Configurações deve usar a API descrita em [SETTINGS.md](docs/SETTINGS.md); segredos de infraestrutura nunca devem ser enviados ao cliente.


## Proteções de uso

- O endpoint `POST /v1/ai/chat` limita cada dispositivo a 12 chamadas por minuto; excedentes recebem HTTP 429 (`rate_limit_exceeded`).
- Tetos rígidos: 50 requisições globais/dia, 25 por app/dia, 10 por dispositivo/dia, 512 tokens de saída, 1.500 caracteres por mensagem e 3.000 caracteres por requisição. A tela administrativa poderá reduzir, mas não elevar esses tetos.
- O modelo efetivo é validado por allowlist no próprio Worker. Atualmente, somente `@cf/google/gemma-4-26b-a4b-it` é aceito; uma configuração de ambiente fora da allowlist não será executada.
- As migrations `0004_minute_rate_limit.sql` e `0005_configurable_rate_limit.sql` criam o contador por minuto. A limpeza agendada remove registros antigos.


## Pesquisa e Estúdio Visual da Garcia

A ferramenta `garcia.research` é executada nativamente pelo Cloudflare Worker em TypeScript. Pesquisa textual e Estúdio Visual usam Workers AI e DuckDuckGo; relatórios, imagens e versões ficam no R2, sessões e metadados no D1 e cache de prompts no KV. O GitHub é usado somente para o código-fonte. Não há GitHub Actions para executar pesquisas nem publicação de artefatos no repositório.

O Estúdio Visual mantém sessões iterativas: a Garcia reaproveita o `session_id`, refina o prompt com o feedback e salva cada versão no R2. O cliente Android exibe as imagens diretamente no histórico da conversa.

O pacote `garcia_research/` permanece como ferramenta Python local e opcional para testes/desenvolvimento; não é executado pelo Worker e não publica artefatos. Consulte [docs/RESEARCH_WORKER_GUIDE.md](docs/RESEARCH_WORKER_GUIDE.md) para configurar R2, D1, KV, modelos e executar `wrangler dev`. Consulte [docs/GARCIA_RESEARCH.md](docs/GARCIA_RESEARCH.md) para o pacote local.
