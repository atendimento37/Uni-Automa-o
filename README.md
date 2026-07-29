# UniFlow — Fases 2 a 7 completas + Fase 8 (Integrações, parcial)

Automação do processo de comissões da UniEmbalagens.

- **Fase 2**: modelagem completa do banco (Supabase/PostgreSQL) e autenticação
  com controle de sessão e permissões por papel.
- **Fase 3**: casca do app (menu lateral, barra superior, dashboard) +
  **módulo de Cadastros completo**: Fornecedores, Clientes, Representantes e
  Regras de Comissão, com listagem, criação, edição e inativação (nunca
  exclusão definitiva — dado histórico é preservado).
- **Fase 4 (Importador Inteligente)**: upload real de Excel/CSV, parsing da
  planilha, mapeamento semântico de colunas via IA (OpenAI), padronização,
  busca/criação automática de cliente e representante, e (já adiantando parte
  da Fase 5) cálculo de comissão pelo motor de regras + comparação com o
  valor informado + criação automática de pendência em caso de divergência.
  **PDF ainda não é suportado como entrada** — fica registrado como limitação
  conhecida, não escondida.
- **Fase 6 (Pendências e Aprovações)**: tela de pendências com atribuição,
  prazo e comentários; aprovação comercial (aprovar / reprovar / solicitar
  ajuste); aprovação financeira (aprovar com valor editável, gerando
  automaticamente um registro de pagamento pendente, ou reprovar). Toda
  decisão fica registrada em `commercial_approvals`/`financial_approvals` para
  auditoria.
- **Pagamentos**: programar data, marcar como pago, gerar comprovante em PDF
  (com a logo real da UniEmbalagens) e exportar a lista em Excel.
- **Fase 7 (Relatórios)**: Comissões, Pendências, Pagamentos, Representantes,
  Fornecedores e Auditoria (este último só visível/exportável para admin —
  reforçado tanto na interface quanto por RLS no banco), todos exportáveis em
  PDF, Excel ou CSV, com filtros por período/fornecedor/representante/cliente/status.

## O que está incluído

- `supabase/migrations/` — 10 migrations, na ordem em que devem ser aplicadas:
  1. Extensões e enums
  2. Perfis de usuário (`profiles`) + trigger de criação automática no cadastro
  3. Cadastros mestres (fornecedores, clientes, representantes, produtos, categorias)
  4. Pedidos, Pedidos SV e notas fiscais/documentos
  5. Motor de regras de comissão + tabela `imports` + `commissions`
  6. Pipeline do Importador Inteligente (registros brutos, templates de layout aprendidos, auditoria de mapeamento)
  7. Pendências, aprovações comercial/financeira e pagamentos
  8. Anexos, notificações e log de auditoria
  9. Trigger genérico de auditoria aplicado às tabelas sensíveis
  10. Row Level Security (RLS) por papel — admin / comercial / financeiro / executivo
- `supabase/seed.sql` — fornecedores já identificados na análise das planilhas reais
- App Next.js (`src/`) com autenticação completa:
  - Login, logout, recuperação de senha, alteração de senha
  - Middleware de sessão e proteção de rotas por papel
- Casca do app (`src/components/app-shell/`):
  - Menu lateral responsivo (vira gaveta em telas estreitas), itens filtrados por papel, logo real da UniEmbalagens
  - Barra superior com busca global (stub), alternância de tema claro/escuro (paleta oficial dourado/grafite), menu do usuário
  - Dashboard com indicadores reais consultados no Supabase (tolerante a banco vazio)
- Módulo de Cadastros (`src/app/(app)/cadastros/`, `src/components/cadastros/`, `src/lib/cadastros/actions.ts`):
  - Fornecedores, Clientes, Representantes e Regras de Comissão — listar, criar, editar
  - "Inativar" em vez de excluir (soft delete/desativação) — nenhum dado histórico é apagado
  - Regras de comissão com seletor de fornecedor/cliente/representante, faixa de valor e prioridade — tudo editável pelo admin sem tocar em código
- Importador Inteligente (`src/app/(app)/importacao/`, `src/app/api/importacao/upload/`, `src/lib/importer/`):
  - Upload de .xlsx/.xls/.csv (PDF é rejeitado com mensagem clara — não implementado ainda)
  - Parser que detecta a linha de cabeçalho mesmo com texto solto acima/abaixo e cabeçalho quebrado em 2 linhas
  - Detecção de fornecedor pelo nome do arquivo (padrão observado: `COMISSÃO_<FORNECEDOR>_<PERÍODO>`)
  - Mapeamento semântico de colunas via OpenAI, com cache por fornecedor (`supplier_layout_templates`) — só chama a IA de novo se o layout mudar
  - Normalização de número/data em formato brasileiro (ex: "R$ 55.800,00", "24/04/2026")
  - Busca ou cria automaticamente cliente e representante por nome
  - Motor de regras: resolve a regra mais específica ativa e calcula a comissão; compara com o valor informado e cria pendência automática se divergir
  - Tela de detalhe mostrando o mapeamento aplicado (com % de confiança) e todos os registros processados

### Variáveis de ambiente adicionais (Importador Inteligente)

```
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini   # opcional, esse é o padrão
```

Sem `OPENAI_API_KEY`, o upload de planilhas falha com erro explícito (não há fallback silencioso).

- Pendências e Aprovações (`src/app/(app)/pendencias/`, `src/app/(app)/aprovacoes/`, `src/lib/pendencias/`, `src/lib/aprovacoes/`):
  - Pendências: filtro por status, atribuir responsável, definir prazo, comentar, resolver/cancelar/reabrir
  - Aprovação Comercial: aprovar (resolve a pendência automaticamente), reprovar, ou solicitar ajuste (reabre/cria pendência com o comentário)
  - Aprovação Financeira: aprovar com valor editável (cria automaticamente um registro em `payments` com status "pendente"), ou reprovar
  - Toda decisão fica registrada em `commercial_approvals`/`financial_approvals`, alimentando a auditoria
- Pagamentos (`src/app/(app)/pagamentos/`, `src/lib/pagamentos/`, `src/app/api/pagamentos/`):
  - Abas por status (pendente/programado/pago), programar data, marcar como pago
  - Comprovante em PDF gerado sob demanda (`pdf-lib`), com a logo real da UniEmbalagens, salvo no bucket `comprovantes` e reaproveitado nos próximos acessos (não regenera a cada clique)
  - Exportação da lista para Excel (`xlsx`)
- Relatórios (`src/app/(app)/relatorios/`, `src/lib/relatorios/`, `src/app/api/relatorios/export/`):
  - 6 tipos de relatório com filtros por período/fornecedor/representante/cliente/status
  - Exportação em PDF (gerador tabular paginado próprio), Excel e CSV a partir da mesma consulta normalizada
  - Auditoria só aparece na interface para admin, e a própria RLS do banco garante que a consulta retorna vazio para qualquer outro papel — dupla proteção
- Integrações (`src/app/(app)/integracoes/`, `src/lib/integracoes/suas-vendas/`, `src/app/api/integracoes/suas-vendas/`):
  - **ERP Suas Vendas**: **API confirmada como existente**, mas seu uso depende de aprovação da diretoria para pagamento da taxa. Enquanto isso, foi implementado o **fallback por arquivo** já previsto no escopo — exporta os Pedidos SV pendentes em Excel, e importa de volta a confirmação (coluna preenchida com "SIM"), marcando `synced_at`/`sync_source`.
  - A camada de acesso usa um **adaptador substituível** (`SuasVendasAdapter`): assim que a taxa for aprovada e a documentação da API for obtida, basta implementar um `ApiSuasVendasAdapter` com a mesma interface e trocar a fábrica em `index.ts` — nenhuma tela muda.
  - **E-mail automático**: não implementado — está listado como `future_features` (desabilitado) no escopo original, não como parte desta fase.

## Como aplicar no seu projeto Supabase

```bash
# 1. Instale a CLI do Supabase, se ainda não tiver
npm install -g supabase

# 2. Conecte ao seu projeto (crie um em supabase.com se necessário)
supabase link --project-ref SEU_PROJECT_ID

# 3. Aplique as migrations, na ordem
supabase db push

# 4. (Opcional) rode o seed de fornecedores
psql "$(supabase db url)" -f supabase/seed.sql
```

## Como rodar o app localmente

```bash
cp .env.local.example .env.local
# preencha NEXT_PUBLIC_SUPABASE_URL e NEXT_PUBLIC_SUPABASE_ANON_KEY
# (encontrados em Project Settings > API no painel do Supabase)

npm install
npm run dev
```

## Criando o primeiro usuário admin

1. Cadastre um usuário normalmente pelo Supabase Auth (painel ou tela de convite futura).
2. Um `profile` é criado automaticamente com papel `commercial` (padrão seguro).
3. Promova manualmente para admin:

```sql
update public.profiles set role = 'admin' where id = 'UUID_DO_USUARIO';
```

## Decisões em aberto (aguardando definição com o time)

- **Tolerância de divergência** entre comissão calculada e informada: hoje
  qualquer diferença ≠ 0 é candidata a pendência. Um valor de tolerância
  configurável poderá ser adicionado em `commission_rules` quando definido
  com o financeiro.
- **Regras de faixa/meta/campanha/bonificação**: a estrutura em
  `commission_rules` já suporta (`min_base_value`, `max_base_value`,
  `priority`), mas nenhuma regra desse tipo foi pré-cadastrada — não há
  evidência delas nas planilhas analisadas.
- **API do ERP Suas Vendas**: confirmado que existe, mas aguardando
  aprovação da diretoria para pagamento da taxa de acesso. Enquanto isso, a
  integração funciona por arquivo Excel (exportar pendentes → importar
  confirmação). Assim que a taxa for aprovada e a UniEmbalagens obtiver a
  documentação da API, a troca para modo API é local (um novo adaptador),
  sem redesenhar nada.

## Próxima fase

Fase 8 concluída na parte que dependia só de nós (Suas Vendas por arquivo).
Faltam: e-mail automático (fora de escopo por ora), Fase 9 (Mobile) e Fase 10
(Produção: testes, segurança, deploy, monitoramento) — todas dependem de
decisões externas antes de eu poder propor arquitetura concreta.
