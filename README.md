# Handit Planning — visão funcional e especificação de integração

> **Status deste documento:** consolidação dos materiais entregues na pasta `documentação/`. Serve como guia de entendimento e preparação da integração; não substitui a validação de layout/ambiente com a Handit. Os arquivos de origem não foram incluídos neste repositório, portanto as referências abaixo apontam para os nomes recebidos, não para links versionados.

## 1. Visão geral

O Handit Planning é uma plataforma de planejamento que recebe estruturas organizacionais, dimensões contábeis, realizados e dados de planejamento para disponibilizá-los nas modelagens da plataforma. Os materiais disponíveis cobrem:

- cadastro de empresas e filiais;
- plano de contas contábil hierárquico;
- centros de custo (hierarquia e atributos);
- razão contábil / realizado;
- orçamento contábil;
- CAPEX e depreciação;
- cargos e quadro de colaboradores;
- estrutura de DRE contábil.

A pasta de origem não descreve uma API pública, autenticação, protocolo de transferência ou um fluxo de implantação automatizado de ponta a ponta. Ela fornece principalmente **layouts de dados** (para arquivo/SQL, conforme o caso) e uma especificação de infraestrutura da plataforma. Por isso, este guia distingue o que está documentado do que ainda precisa ser confirmado.

## 2. Como a integração deve ser entendida

A integração deve entregar os dados nos layouts acordados com a Handit, mantendo nomes de campos, tipos, referências e granularidade consistentes. Os materiais mencionam arquivo e SQL, mas não definem o mecanismo de conexão, o formato final do arquivo nem o comportamento de carga. O desenho abaixo é um fluxo recomendado para organizar a implementação — não uma descrição de um produto/API já especificado:

1. **Extrair** os cadastros e movimentos do ERP, sistema contábil, RH e fontes de orçamento.
2. **Transformar** os dados para os layouts Handit, preservando códigos como identificadores textuais quando puderem conter zeros à esquerda.
3. **Validar** campos obrigatórios, datas, valores, domínio de códigos e vínculos entre cadastros e movimentos.
4. **Entregar** pelo canal, conexão e agenda definidos com a equipe Handit.
5. **Conferir o processamento** e reconciliar contagens e totais com a origem. A forma de consultar logs/retornos precisa ser definida para cada implantação.

### Dependências recomendadas entre cargas

Carregue primeiro as dimensões referenciadas e depois os movimentos:

1. empresas/filiais;
2. plano de contas e centros de custo (hierárquicos e atributos);
3. cargos e estrutura DRE, quando utilizados;
4. razão contábil, orçamento, CAPEX e RH por colaborador.

Essa ordem é uma recomendação baseada nas referências entre os layouts. A documentação não informa se o sistema exige uma ordem específica, se rejeita registros sem dimensão correspondente ou se permite carga parcial.

### Regras de preparação comuns

- Os campos estão listados mais adiante como obrigatórios ou opcionais conforme as planilhas. Para campos opcionais, mantenha a coluna no layout e envie vazia quando não houver valor, conforme a orientação explícita do layout do razão.
- Códigos de empresa, filial, conta, centro de custo, cargo e colaborador são descritos como texto ou número livre em vários layouts. Trate-os como **identificadores**, não como quantidades: não arredonde, não converta para notação científica e não elimine zeros à esquerda.
- Datas são descritas em geral como `DD/MM/AAAA`; as planilhas de exemplo também contêm datas que o Excel exibe como data/hora. Confirme com a Handit o formato efetivamente aceito pelo canal de carga.
- Valores contábeis são descritos com duas casas decimais. Confirme separador decimal e convenção de milhar antes de produzir arquivos; os exemplos usam ponto decimal em algumas células.
- Preserve os registros no grão da origem. As planilhas não especificam chaves de negócio, política de atualização, deduplicação, exclusão lógica nem se a carga é incremental ou substitutiva. Defina esses pontos antes de agendar cargas recorrentes.

## 3. Catálogo dos layouts disponíveis

### 3.1 Empresas e filiais

**Objetivo:** informar as empresas e respectivas filiais.

| Campo | Descrição | Obrigatório |
|---|---|---:|
| `EMPRESA_COD` | Código da empresa | Sim |
| `EMPRESA_DESC` | Nome da empresa | Sim |
| `EMPRESA_COD + FILIAL_COD` | Código da filial (a definição combina empresa e filial) | Sim |
| `FILIAL_DESC` | Nome da filial | Sim |

**Observação:** o identificador combinado aparece assim na definição do campo; em várias abas de exemplo, a coluna é chamada simplesmente `FILIAL_COD`. Confirme o nome e a regra de composição esperados no ponto de integração.

### 3.2 Plano de contas hierárquico

| Campo | Descrição | Formato / domínio | Obrigatório |
|---|---|---|---:|
| `CONTA_COD_RED` | Código reduzido da conta contábil | Texto ou número livre | Sim |
| `CONTA_CLASSIFICACAO` | Classificação da conta | Texto ou número livre | Sim |
| `CONTA_DESC` | Descrição da conta | Texto ou número livre | Sim |
| `NIVEL` | Nível da conta | Número livre | Sim |
| `ANA_SIN` | Conta analítica ou sintética | `A` ou `S` (`A` = analítica; `S` = sintética) | Sim |
| `CONTA_PAI` | Código reduzido da conta pai | Texto ou número livre | Sim |

O exemplo de origem usa, entre outros, código reduzido `14529`, classificação `1.11.111.1111`, nível `6`, tipo `A` e conta pai `14300`. A forma de representar a conta raiz quando não existe pai não está definida; confirme se deve ser vazio, zero ou outro valor.

### 3.3 Centro de custo — atributos

| Campo | Descrição na planilha | Obrigatório |
|---|---|---:|
| `UNIDADE_ID` | Código da unidade | Sim |
| `UNIDADE_DESC` | Nome da unidade | Sim |
| `DIRETORIA_ID` | Código da diretoria | Sim |
| `DIRETORIA_DESC` | Nome da diretoria | Sim |
| `ÁREA_ID` | Código da área | Sim |
| `ÁREA_DESC` | Nome da área | Sim |
| `CC_DESC` | A planilha descreve como código reduzido do centro de custo | Sim |
| `CC_COD_RED` | A planilha descreve como descrição do centro de custo | Sim |

**Inconsistência da fonte:** as descrições/exemplos dos dois últimos campos parecem invertidos: `CC_DESC` traz exemplo semelhante a um código (`11002`) e `CC_COD_RED` traz exemplo descritivo (`11002 - Administrativo Industrial`). A interpretação provável pelos nomes é `CC_COD_RED` = código e `CC_DESC` = descrição, mas isso deve ser confirmado com a Handit antes do mapeamento. Os nomes de área também aparecem com acento (`ÁREA_ID`, `ÁREA_DESC`); confirme se o layout de entrada exige exatamente essa grafia.

### 3.4 Centro de custo — hierárquico

| Campo | Descrição | Formato / domínio | Obrigatório |
|---|---|---|---:|
| `CC_COD_RED` | Código reduzido do centro de custo | Texto ou número livre | Sim |
| `CC_CLASSIFICACAO` | Classificação do centro de custo | Texto ou número livre | Sim |
| `CC_DESC` | Descrição do centro de custo | Texto ou número livre | Sim |
| `NIVEL` | Nível do centro de custo | Número livre | Sim |
| `ANA_SIN` | Centro de custo analítico ou sintético | A planilha diverge: formato `A` ou `N`, observação `A` ou `S` | Sim |
| `CC_PAI` | Código reduzido do centro de custo pai | Texto ou número livre | Sim |

**Inconsistência da fonte:** o domínio de `ANA_SIN` conflita entre `A/N` e `A/S`; não escolha um deles sem validação. A representação do centro de custo raiz também não está especificada.

### 3.5 Razão contábil

Cada linha representa um lançamento contábil no layout de origem. Os valores de rateio devem chegar **já rateados**. Quando o lançamento não tiver centro de custo, a planilha orienta enviar `CC_COD` com o padrão `0`.

| Campo | Descrição | Obrigatório | Regra documentada |
|---|---|---:|---|
| `DATA_LANCAMENTO` | Data do lançamento | Sim | `DD/MM/AAAA` |
| `EMPRESA_COD` | Código da empresa | Sim | Texto/número livre |
| `FILIAL_COD` (definição: `EMPRESA_COD + FILIAL_COD`) | Código da filial | Sim | Confirmar nome/composição com a Handit |
| `CC_COD` | Código do centro de custo | Sim | Enviar `0` se não houver CC |
| `CONTA_CONTABIL_COD` | Conta contábil | Sim | Código reduzido ou classificação/estrutura |
| `NATUREZA_LANCAMENTO` | Natureza do lançamento | Sim | `D` = débito; `C` = crédito |
| `VALOR_LANCAMENTO` | Valor do lançamento | Sim | Número com duas casas; já rateado |
| `LOTE_LANCAMENTO` | Lote | Não | Manter a coluna vazia se ausente |
| `NUMERO_LANCAMENTO` | Número do lançamento | Não | Manter a coluna vazia se ausente |
| `SEQUENCIA_LANCAMENTO` | Sequência | Não | Manter a coluna vazia se ausente |
| `HISTORICO_LANCAMENTO` | Histórico/complemento | Não | Manter a coluna vazia se ausente |
| `FOR_CLI_DESC` | Fornecedor/cliente | Não | Manter a coluna vazia se ausente |
| `UNIDADE_NEGOCIO` | Unidade de negócio | Não | Manter a coluna vazia se ausente |
| `ORIGEM` | Módulo de origem | Não | Manter a coluna vazia se ausente |
| `CONTA_CONTABIL_DEB` | Conta contábil do débito | Não | Campo informativo para lançamentos débito/crédito |
| `CONTA_CONTABIL_CRE` | Conta contábil do crédito | Não | Campo informativo para lançamentos débito/crédito |
| `CC_DEB` | Centro de custo do débito | Não | Campo informativo para lançamentos débito/crédito |
| `CC_CRE` | Centro de custo do crédito | Não | Campo informativo para lançamentos débito/crédito |
| `ITEM_CONTABIL_DEB` | Item contábil do débito | Não | Campo informativo para lançamentos débito/crédito |
| `ITEM_CONTABIL_CRE` | Item contábil do crédito | Não | Campo informativo para lançamentos débito/crédito |
| `ITEM_CONTABIL` | Item contábil | Não | Manter a coluna vazia se ausente |

A observação do layout diz que, quando o tipo de lançamento for ambos (débito e crédito), os lançamentos devem vir duplicados: uma linha para crédito e outra para débito. Os campos auxiliares de débito/crédito são descritos como informativos. Confirme com a Handit como preencher `NATUREZA_LANCAMENTO`, `VALOR_LANCAMENTO` e os campos auxiliares nessas duas linhas para evitar duplicar valores no realizado.

**Exemplo ilustrativo** (separador `;` apenas para facilitar a leitura; não é uma especificação de transporte):

```csv
DATA_LANCAMENTO;EMPRESA_COD;FILIAL_COD;CC_COD;CONTA_CONTABIL_COD;NATUREZA_LANCAMENTO;VALOR_LANCAMENTO;LOTE_LANCAMENTO;NUMERO_LANCAMENTO;SEQUENCIA_LANCAMENTO;HISTORICO_LANCAMENTO;FOR_CLI_DESC;UNIDADE_NEGOCIO;ORIGEM;CONTA_CONTABIL_DEB;CONTA_CONTABIL_CRE;CC_DEB;CC_CRE;ITEM_CONTABIL_DEB;ITEM_CONTABIL_CRE;ITEM_CONTABIL
01/01/2025;1;1.1;1005;5000;D;105.50;1;1012025001;1;Fornecedor materiais;;;;;;;;;;
```

Os códigos e o valor acima seguem o padrão de exemplo da planilha; o separador, representação de data e campos de crédito/débito precisam ser adequados ao canal efetivamente contratado.

### 3.6 Orçamento contábil

Uma linha representa o valor orçado por cenário, período, filial, conta e centro de custo.

| Campo | Descrição | Obrigatoriedade na fonte |
|---|---|---:|
| `CENARIO` | Descrição do cenário | Sim |
| `ANO` | Ano (`AAAA`) | Sim |
| `MES` | Mês (`MM`) | **Não informado explicitamente** |
| `FILIAL_COD` (definição: `EMPRESA_COD + FILIAL_COD`) | Filial | Sim |
| `CONTA_COD_RED` | Código reduzido da conta | Sim |
| `CC_COD_RED` | Código reduzido do centro de custo | Sim |
| `VLR_ORCADO` | Valor orçado | Sim, número com duas casas decimais |

**Ponto a validar:** `MES` tem exemplo, mas a célula de obrigatoriedade está vazia na planilha. O nome da filial também varia entre a definição composta e o cabeçalho de exemplo `FILIAL_COD`.

**Exemplo ilustrativo baseado no cabeçalho da planilha:**

```csv
CENARIO;ANO;MES;FILIAL_COD;CONTA_COD_RED;CC_COD_RED;VLR_ORCADO
Orçamento 2025;2025;1;1.1;31001;11001;1250.00
```

### 3.7 CAPEX

Cada linha descreve um bem e os dados usados para o cálculo/planejamento de depreciação.

| Campo | Descrição | Obrigatório na revisão da planilha |
|---|---|---:|
| `DATA_INICIO` | Data de início da depreciação | Não |
| `DATA_FIM` | Data fim | Sim |
| `EMPRESA_COD` | Empresa que adquiriu o bem | Sim |
| `FILIAL_COD` (definição: `EMPRESA_COD + FILIAL_COD`) | Filial do bem | Sim |
| `CC_COD` | Centro de custo do bem | Sim |
| `GRUPO_ATIVO_COD` | Código do grupo de ativo | Sim |
| `GRUPO_ATIVO_DESC` | Descrição do grupo de ativo | Sim |
| `BEM_COD` | Código do bem | Não |
| `BEM_DESC` | Descrição do bem | Não |
| `VLR_DEPRECIACAO_MES` | Valor mensal de depreciação | Sim |
| `VLR_ORIGINAL` | Valor do bem/aquisição | Não |
| `FILIAL_AQUISICAO_COD` | Filial de aquisição | Não |

A planilha tem uma coluna de obrigatoriedade original e outra “revisado Maiola”; a tabela acima usa a coluna revisada. Há alterações em relação ao status original, inclusive `DATA_INICIO`, `BEM_COD`, `BEM_DESC` e `VLR_ORIGINAL`. Confirme qual revisão está vigente. As datas aparecem como `DD/MM/AAAA`; exemplos incluem datas numéricas do Excel. Um exemplo de valores do layout é grupo `Máquinas e Equipamentos`, depreciação mensal `200` e valor original `12000`.

### 3.8 Cargos de RH

| Campo | Descrição | Obrigatório |
|---|---|---:|
| `EMPRESA_COD` | Código da empresa | Sim |
| `EMPRESA_DESC` | Nome da empresa | Sim |
| `FILIAL_COD` (definição: `EMPRESA_COD + FILIAL_COD`) | Código da filial | Sim |
| `FILIAL_DESC` | Nome da filial | Sim |
| `CARGO_COD` | Código do cargo | Sim |
| `CARGO_DESC` | Descrição do cargo | Sim |

A aba de exemplo usa os nomes `ID_EMPRESA`, `DESC_EMPRESA`, `ID_FILIAL`, `DESC_FILIAL`, `CARGO_COD` e `CARGO_DESC`, que diferem dos nomes definidos na aba de campos. Valide qual cabeçalho a integração deve enviar; não presuma que os nomes de exemplo sejam aceitos como aliases.

### 3.9 RH por colaborador

| Campo | Descrição | Obrigatório | Observação |
|---|---|---:|---|
| `DATA_BASE` | Data de referência do quadro | Sim | A fonte informa `DD/MM/AAAA` |
| `FILIAL_COD` (definição: `EMPRESA_COD + FILIAL_COD`) | Filial | Sim | Confirmar nome/composição |
| `CC_COD` | Centro de custo | Sim | — |
| `CARGO_COD` | Código do cargo | Sim | Se não houver código, repetir a descrição do cargo |
| `CARGO_DESC` | Descrição do cargo | Sim | — |
| `COLABORADOR_COD` | Código do colaborador | Sim | — |
| `COLABORADOR_DESC` | Descrição/nome do colaborador | Sim | — |
| `VLR_SALÁRIO` | Valor do salário | Sim | Confirmar se o nome com acento é literal no layout |
| `QTD_VIDAS` | Quantidade de vidas | Não | Informar `0` caso não haja |

A planilha de exemplo usa `FILIAL_COD` e `VLR_SALÁRIO`. Como se trata de dado pessoal e salarial, aplique controles de acesso, transmissão e retenção definidos pela organização; os exemplos deste documento não reproduzem nomes de pessoas da planilha.

### 3.10 Estrutura DRE contábil

Cabeçalho identificado na aba `DRE`:

| Campo | Finalidade aparente |
|---|---|
| `TIPO_LINHA` | Identifica a linha como `CONTA` ou `FÓRMULA` |
| `LINHA_DRE_N1_ID` | Identificador do nível 1 da DRE |
| `LINHA_DRE_N1_DESC` | Descrição do nível 1 |
| `LINHA_DRE_N2_ID` | Identificador do nível 2 |
| `LINHA_DRE_N2_DESC` | Descrição do nível 2 |
| `CONTA_CONTABIL_ANA_COD` | Código da conta analítica associada |
| `CONTA_CONTABIL_ANA_DESC` | Descrição da conta associada |

A amostra associa contas contábeis a linhas da DRE e possui linhas `FÓRMULA` (por exemplo, receita líquida e lucro bruto). A planilha não fornece a expressão das fórmulas nem um dicionário completo de níveis/regras de cálculo. Ela também contém linhas com campos vazios. A regra de interpretação e cálculo das fórmulas precisa ser obtida com o responsável pela modelagem/Handit antes de automatizar a carga.

## 4. Exemplos de validação antes da carga

Implemente uma validação prévia que, no mínimo:

- rejeite ou sinalize campos obrigatórios ausentes;
- valide `NATUREZA_LANCAMENTO` contra `D`/`C` e `ANA_SIN` contra o domínio acordado para cada layout;
- confira datas válidas e período contábil esperado;
- valide valores numéricos e escala de duas casas quando aplicável;
- confira se filial, conta, centro de custo e cargo referenciados existem nos respectivos cadastros;
- garanta que campos opcionais não desapareçam do cabeçalho/layout quando estiverem vazios;
- compare quantidade de linhas e soma de valores da origem com os dados preparados, principalmente para razão e orçamento;
- registre lote, horário, origem e resultado de cada execução no processo de integração.

Essas são recomendações de controle operacional; as planilhas não especificam o mecanismo de retorno, política de erros ou monitoramento da importação.

## 5. Especificação técnica de infraestrutura

As informações abaixo foram transcritas do PDF **“Especificação plataforma Handit Planning v7”**. O documento declara considerar uma quantidade média de consumo de modelagens e dados integrados. Confirme se ainda é a versão vigente e se o dimensionamento atende ao volume real do ambiente.

### Servidor de aplicação

| Recurso / componente | Requisito registrado |
|---|---|
| CPU | Em caso de virtualização, mínimo de 8 processadores lógicos alocados |
| Memória | 32 GB RAM |
| Disco | 150 GB |
| Sistema operacional | Ubuntu Server LTS 22.04 ou Microsoft Windows Server “2016 Server R2 ou superior” com service packs atualizados (texto conforme PDF; confirmar edição/versão exata) |
| Privilégio | O usuário do sistema operacional deve possuir permissão de administrador no servidor disponibilizado |

O PDF não explicita particionamento, política de crescimento, backup, alta disponibilidade, firewall, portas ou ambientes separados (desenvolvimento/homologação/produção).

### Software e bancos de dados

| Item | Versão / observação da especificação |
|---|---|
| JDK | Versões 8, 11 e 17 |
| Navegadores cliente | Google Chrome ou Mozilla Firefox |
| Outros componentes | NGINX, Rundeck e Git |
| Banco relacional | Oracle 10g R2 ou superior **ou** SQL Server versão 17 ou superior |
| Restrição SQL Server | SQL Server Express não é aceito |
| MongoDB | Versão 4.4.0 |

Segundo o PDF, o banco Oracle ou SQL Server armazena tabelas e dados do Planning; MongoDB armazena logs e fluxo de persistência. “SQL Server versão 17 ou superior” foi mantido como aparece na fonte, mas a nomenclatura precisa ser confirmada com a Handit para evitar ambiguidade.

### Componentes da plataforma

A arquitetura é descrita em duas camadas: **cliente** (estações de trabalho com sistema operacional e navegador, sem necessidade de processamento pesado) e **infraestrutura** (servidores, rede, aplicações e bancos de dados). O PDF lista quatro serviços no servidor de aplicação, junto com um repositório:

- Handit Planning;
- Handit Data Integration;
- Handit Proxy NGINX;
- Rundeck Integration;
- repositório (o documento também lista Git entre os softwares).

## 6. Pontos que precisam de confirmação com a Handit

Antes de desenvolver ou colocar a integração em produção, fechar por escrito:

1. canal suportado para cada carga (arquivo, banco/SQL ou outro), conectividade, autenticação, criptografia, rede/portas e responsabilidade por credenciais;
2. formatos aceitos, extensão, delimitador, codificação, convenção de datas/decimais, tratamento de aspas e valores nulos;
3. cabeçalhos canônicos, especialmente filial composta vs. `FILIAL_COD`, cargos RH (cabeçalho de definição vs. exemplo), `VLR_SALÁRIO` e campos com acentos;
4. regra para registros raiz sem pai e valores válidos de `ANA_SIN` nos dois layouts hierárquicos;
5. obrigatoriedade de `MES` no orçamento e revisão vigente do layout CAPEX;
6. semântica das duas linhas de lançamento débito/crédito e campos auxiliares do razão;
7. chaves de negócio, comportamento de reenvio, atualização/exclusão, carga completa ou incremental, idempotência e tratamento de duplicidades;
8. limites de volume, frequência/janela, timeout, mecanismo de retorno e localização dos logs/erros;
9. regras/calculadoras das linhas `FÓRMULA` da DRE;
10. versões vigentes e sizing definitivo de SO, bancos, JDK, serviços, CPU, RAM e disco.

## 7. Materiais de origem analisados

Os itens abaixo estavam na pasta `documentação/` recebida para esta consolidação:

1. `1 - Campos da Estrutura Empresarial.xlsx`
2. `2 - Campos do Plano de Contas Hierárquico - ORI.xlsx`
3. `3-1 - Campos do Centro de Custo - Atributos.xlsx`
4. `3-2 - Campos do Centro de Custo - Hierárquico.xlsx`
5. `4 - Campos do Razão Contábil.xlsx`
6. `5 - Campos da Carga de Orçamento Contábil.xlsx`
7. `6 - Campos CAPEX.xlsx`
8. `7 - Campos Cargos RH.xlsx`
9. `8 - Campos do RH por Colaborador.xlsx`
10. `9 - Estrutura DRE Contábil.xlsx`
11. `Especificação plataforma Handit Planning v7.pdf`

A pasta também contém abas de exemplo para os layouts. Foram usadas para compor exemplos e identificar diferenças em relação aos cabeçalhos/definições; divergências não foram ocultadas e estão marcadas neste documento para validação.
