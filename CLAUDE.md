# COMPRAS - CENTRO DE DISTRIBUIÇÃO/AR

Sistema web de compras do Centro de Distribuição / Cozinha de Produção do Grupo AR.
Atende os restaurantes **Arvo, Terra e Voar**. Dois módulos: **Compras** (alimentos) e **Material de Limpeza**.

## Quem usa e como conversar

- O dono do projeto (Marcio) **não é programador**. Explique em português simples, sem jargão.
- Antes de mudar algo visual, **mostre uma simulação/mockup e espere aprovação**. Ele pede isso com frequência.
- Faça uma mudança de cada vez e diga, em poucas linhas, o que mudou e como testar.
- **Regra importante:** o sistema nunca deve alterar dados sozinho. Toda alteração (nomes de fornecedor, valores, status etc.) só acontece por ação explícita do usuário. Nada de "propagar" mudanças para outros itens automaticamente.
- Nunca peça nem aceite tokens/senhas coladas no chat.

## Arquitetura

- **Frontend:** um único arquivo HTML com todo o CSS e JavaScript dentro (~3.600 linhas). Sem framework, sem build.
- **Banco:** Supabase (Postgres), acessado direto do navegador com a chave pública (anon) via `supabase-js`. A URL e a chave ficam nas constantes `SUPA_URL` e `SUPA_KEY`, perto do início do script.
- **Hospedagem:** GitHub Pages, repositório `marciovsc/compras-grupoar`, branch `main`.
  URL: https://marciovsc.github.io/compras-grupoar/
  (Cópia, com todo o histórico, do repositório original `marciocavalcantiar-cd/compras-grupo-ar`. Os dois usam o mesmo Supabase.)
- **Arquivos do repositório:** `index.html` (principal), `app.html` (cópia idêntica, deve ser mantida igual ou removida), `cotacao.html` (página pública que o fornecedor abre pelo link da cotação) e `.nojekyll` (evita falha de build do Jekyll; não apagar).
- Dados ficam em memória no objeto global `C` (`C.pedidos`, `C.itens`, `C.semanas`, `C.setores`, `C.locais_entrega`, etc.) e são recarregados por `reloadAll()`. Existe também sincronização automática a cada 60 segundos (`setInterval` + `refreshDados`).

## Banco de dados (Supabase)

O esquema abaixo foi reconstruído a partir do código; confira no painel do Supabase antes de confiar em tipos exatos.

**Compras:** `restaurantes`, `semanas` (nome, data_inicio, data_fim), `categorias`, `produtos`, `pedidos`, `pedido_itens`, `historico_precos`, `semanas_fechadas` (snapshot usado no Relatório Mensal).

**Material de Limpeza (tabelas separadas):** `semanas_limpeza`, `categorias_limpeza`, `produtos_limpeza`, `pedidos_limpeza`, `itens_limpeza`, `fechamentos_limpeza`.

**Listas editáveis pelo usuário:** `setores` (hoje: Cozinha Quente, Confeitaria) e `locais_entrega` (hoje: Arvo, Voar, Terra, Porto Fino).

Colunas adicionadas ao longo do projeto:
- `pedido_itens`: `previsao_entrega` (date), `setor` (text), `local_entrega` (text)
- `pedidos`: `observacao` (text), `local_entrega` (text)
- `pedidos_limpeza`: `observacao` (text)
- `itens_limpeza`: `previsao_entrega` (date)

Pontos de atenção:
- O fornecedor, a categoria, o setor e o local são guardados **como texto** em cada item (não por ID).
- Status do item: `Aguardando`, `Previsto`, `Entregue`.
- `historico_precos.pedido_item_id` precisa aceitar nulo (itens de limpeza não usam esse vínculo). Itens de limpeza entram no histórico com `restaurante_nome = 'Limpeza'`.
- Todas as tabelas têm RLS ligado com política "permitir tudo" e não há login. Veja "Melhorias sugeridas".
- **Mudanças de banco (SQL) são rodadas pelo Marcio no SQL Editor do Supabase.** Entregue o SQL pronto, peça confirmação de que rodou e só então publique o código que depende dele.

## Regras de negócio e convenções

- Interface toda em português do Brasil.
- Nome da semana padrão: `COMPRAS CD - AR` (sempre em maiúsculas). O período aparece em destaque como `dd/mm a dd/mm`, **sem o ano**. Helpers: `fmtD`, `semPeriodo`, `semLabel`.
- Nomes de restaurantes aparecem em **MAIÚSCULAS** (etiqueta colorida via `restBadgeHtml`). Cores: Arvo `#4A6741`, Terra `#8B5E3C`, Voar `#6B2D4A`.
- Valores em reais com "R$" na mesma linha do número (`white-space:nowrap`).
- Setores e locais vêm de `getSetores()` e `getLocais()`. Os dropdowns têm a opção "+ Cadastrar novo..." (`cadastrarNovoSetor`, `cadastrarNovoLocal`).
- Modais: `showModal(html, {wide:true})` para janelas largas (Novo Pedido usa a larga). Clicar fora fecha o modal, então preservar a seleção de itens (`ncSel`) é importante.

## Recibo do pedido (para o fornecedor)

- Abre pelo botão **Recibo** do card da semana (ou pelo ícone no item). Escolhe o **fornecedor** em uma lista com os fornecedores da semana e filtra por **status, setor e local de entrega**.
- Junta todos os itens da semana com aquele fornecedor, **independente de categoria ou de qual pedido** o item veio. A comparação do nome ignora maiúsculas/minúsculas e espaços nas pontas.
- Separa por restaurante, cada um com o cabeçalho **"NOTA FISCAL EM NOME DE: RESTAURANTE"** e subtotal próprio.
- Tem campo editável de "Local de entrega" no topo, fornecedor em maiúsculas, tabela sem quebra de linha e impressão em **A4 paisagem**.

## Cotação de fornecedores

- Botão **Cotação** no card da semana. Tabelas com prefixo `cot_`: `cot_fornecedores` (cadastro com WhatsApp), `cot_rodadas` (cada cotação, status Aberta/Fechada), `cot_itens` (`cotar` = vai para cotação; `fornecedor_escolhido`/`preco_escolhido` só gravados quando o usuário clica no preço), `cot_convites` (um por fornecedor, com `codigo` do link) e `cot_precos`. Coluna `produtos.fornecedor_fixo` (opcional).
- Já existia no banco uma tabela `cotacoes` que não é do sistema. Não usar nem mexer.
- O fornecedor responde em `cotacao.html#CODIGO`, que só usa as funções `cot_ver` e `cot_responder` (security definer) e vê apenas a própria cotação.
- "Usar na compra" leva os itens escolhidos e os de fornecedor fixo para o **Montar compra** (passo de dividir entre as lojas).

## Espelho de recebimento e relatórios

- **Espelho de Compras** (botão Espelho da semana): filtros de status, setor e local. Colunas de conferência (Conf. e Qtd. recebida). Regras de impressão: cabeçalho da tabela repete, linhas não se cortam entre páginas, cabeçalho de fornecedor nunca fica sozinho no fim da página.
- **Espelho Geral**, **Relatório de Compras** (por restaurante, com status e setor), **Relatório Financeiro** e **Relatório Mensal** (usa `semanas_fechadas`).
- **Material de Limpeza:** semanas, pedidos por fornecedor, previsão de entrega, histórico de preços (mostra "Último preço" ao montar o pedido), Espelho de Recebimento, botão "Gerar relatório" consolidado por período e PDF de pedido ao fornecedor **sem preços** (só medida, nome e quantidade).
- O Novo Pedido mostra o **Total do pedido** em tempo real e tem campo de descrição (editável depois pelo ícone de nota).

## Como validar e publicar

1. Depois de qualquer edição, valide a sintaxe do JavaScript (extrair o conteúdo da tag `<script>` e rodar `node --check`).
2. Se `app.html` continuar existindo, mantenha-o **idêntico** ao `index.html`.
3. Publique por **Pull Request para `main`**. O GitHub Pages atualiza sozinho em ~1 minuto. Se o navegador mostrar versão antiga, é cache (Ctrl+Shift+R).
4. Cuidado com substituições de texto em lote: o arquivo tem trechos muito parecidos (os três espelhos compartilham CSS quase idêntico). Confira que cada alteração atingiu só o lugar certo.

## Pendências e problemas conhecidos

- **Bug não resolvido:** o Marcio relatou que, ao colar nomes de fornecedor em vários itens da tabela de pedidos, alguns itens "mudavam sozinhos" para outro fornecedor (exemplo: Farinha Panko e Gergelim Branco voltavam para "Ceasa Mais"). Já verificado: as gravações funcionam e o valor persiste após recarregar a página. **Não foi confirmado** se a troca acontece sem recarregar ou após cerca de 1 minuto. Suspeitas: corrida entre a edição (`onchange`) e a sincronização automática de 60 s que redesenha a tabela, ou colar em campos enquanto a tela é redesenhada. Investigar sem alterar dados automaticamente.
- Itens criados antes das colunas Setor/Local existirem ficam com "—" até serem editados manualmente.

## Melhorias sugeridas

- Dividir o HTML único em arquivos menores (CSS, JS por módulo) e remover a duplicação `app.html`.
- Ativar login (Supabase Auth) e trocar as políticas "permitir tudo" por regras reais de acesso.
- Rodar verificação automática de sintaxe antes de publicar (GitHub Actions).
- Revisar o recálculo e a atualização automática de 60 s para nunca sobrescrever uma edição em andamento.

## Cópia para outra pessoa

Existe uma versão separada do sistema, **só com o restaurante Arvo**, preparada para uma colega: `copia_colega.html` (credenciais do Supabase trocadas por marcadores) e `schema_novo_projeto.sql` (estrutura das tabelas). Ela é um projeto independente, com banco próprio, e **não está neste repositório**. Mudanças feitas aqui não chegam lá automaticamente.
