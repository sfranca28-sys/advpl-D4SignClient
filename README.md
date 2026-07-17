# Protheus × D4Sign — Assinatura Digital de Recibos de Férias

Integração entre o módulo de **Recursos Humanos do TOTVS Protheus** (Gestão de Férias — rotina GP130) e a plataforma de assinatura digital **[D4Sign](https://d4sign.com.br)**.

O objetivo é automatizar o ciclo completo de geração, envio, assinatura e armazenamento dos recibos de férias dos colaboradores, eliminando a impressão e o recolhimento de assinaturas físicas.

> **Versão Protheus:** 12.2410 · **Autor:** Silvano Franca — ALWA · **Última atualização:** Abril/2026

---

## Índice

- [Visão Geral](#visão-geral)
- [Arquitetura e Componentes](#arquitetura-e-componentes)
- [Fluxo do Processo](#fluxo-do-processo)
- [Componentes em Detalhe](#componentes-em-detalhe)
  - [PE_GP030MNU — Ponto de Entrada](#pe_gp030mnu--ponto-de-entrada)
  - [CAD4S000 — Classe D4SignClient](#cad4s000--classe-d4signclient)
  - [CAD4S001 — Geração do Recibo em PDF](#cad4s001--geração-do-recibo-em-pdf)
  - [CAD4S003 — Envio para Assinatura](#cad4s003--envio-para-assinatura)
  - [CAD4S004 — Download dos Assinados](#cad4s004--download-dos-assinados)
  - [CAD4S005 — Reenvio de Documentos com Erro](#cad4s005--reenvio-de-documentos-com-erro)
- [Convenção de Nome dos Arquivos PDF](#convenção-de-nome-dos-arquivos-pdf)
- [Estrutura de Diretórios](#estrutura-de-diretórios)
- [Parâmetros do Sistema (MV_)](#parâmetros-do-sistema-mv_)
- [Ciclo de Vida do Documento (RH_D4SIGN)](#ciclo-de-vida-do-documento-rh_d4sign)
- [Dicionário de Dados — Campos Customizados](#dicionário-de-dados--campos-customizados)
- [Instalação e Configuração](#instalação-e-configuração)
- [Considerações Técnicas e Boas Práticas](#considerações-técnicas-e-boas-práticas)
- [Referências](#referências)

---

## Visão Geral

O processo integrado realiza, de forma encadeada:

1. Geração do recibo de férias em **PDF** diretamente no Protheus (via `FWMsPrinter`).
2. **Upload** do PDF para o cofre seguro do D4Sign.
3. Identificação e cadastro do colaborador como **signatário** do documento.
4. Disparo do **e-mail de assinatura** ao colaborador.
5. **Monitoramento** do status de assinatura.
6. **Download** do documento assinado e armazenamento no servidor.
7. Atualização do **status** do registro no banco de dados do Protheus (tabela `SRH`).

## Arquitetura e Componentes

| Arquivo | Componente | Descrição |
|---|---|---|
| `PE_GP030MNU.tlpp` | Ponto de entrada | Adiciona o botão **"Impressão d4Sign"** na rotina de férias (GP130). |
| `CAD4S000.tlpp` | Classe `D4SignClient` | Camada de comunicação e autenticação com a API REST do D4Sign. |
| `CAD4S001.PRX` | Geração do PDF | Gera o recibo de férias em PDF usando `FWMsPrinter` (substitui a impressão padrão do GP130). |
| `CAD4S003.tlpp` | Job de envio | Upload dos PDFs pendentes, cadastro do signatário e envio para assinatura. |
| `CAD4S004.tlpp` | Job de download | Monitoramento dos documentos enviados e download dos recibos assinados. |
| `CAD4S005.tlpp` | Job de reenvio | Reprocessa documentos com erro na adição do signatário ou no envio para assinatura. |
| `U_CAD4S001.ch` | Include | Definições de strings i18n (`FWI18NLang`) utilizadas pelo CAD4S001. |
| `U_CAD4S001_pt-br.tres` | Recurso i18n | Textos em português dos recibos (aviso de férias, abono, 13º etc.). |

## Fluxo do Processo

```
┌─────────────────────────────────────────────────────────────────┐
│ ETAPA 1 — Geração do PDF (manual, operador de RH)               │
│ GP130 → botão "Impressão d4Sign" (PE_GP030MNU) → u_CAD4S001     │
│ → PDF gravado em <CS_D4SDIR>\pendente\                          │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ ETAPA 2 — Envio para assinatura (CAD4S003, job agendado)        │
│ Varre pendente\ → UploadDocument → consulta SRA →               │
│ AddSigner → SendToSign → SRH.RH_D4SIGN = '2' →                  │
│ move PDF para enviado\<empresa>\                                │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ ETAPA 3 — Download dos assinados (CAD4S004, job agendado)       │
│ Consulta SRH (status '1'/'2') → DownloadSigned → baixa ZIP →    │
│ extrai em assinado\<empresa>\ → SRH.RH_D4SIGN = '4'             │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ CONTINGÊNCIA — Reenvio de erros (CAD4S005, job agendado)        │
│ Varre erro\addsigner\ e erro\sendsigner\ → reexecuta            │
│ AddSigner / SendToSign → SRH.RH_D4SIGN = '2'                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Componentes em Detalhe

### PE_GP030MNU — Ponto de Entrada

**Arquivo:** `PE_GP030MNU.tlpp`

Ponto de entrada padrão do Protheus executado durante a inicialização do menu da rotina **GP130** (Impressão de Recibos de Férias). Adiciona o botão **"Impressão d4Sign"** à barra de ações, vinculado à função `u_CAD4S001`.

A variável pública `aRotina` contém a lista de opções do menu; o PE utiliza `AAdd` para inserir a nova entrada sem remover as demais (o código de remoção do botão padrão está comentado e pode ser ativado, se necessário).

```xbase
User Function GP030MNU()
    Local aArea := GetArea()

    // Adiciona o botão de impressão para a nova rotina de férias
    AAdd(ARotina, { "Impressão d4Sign", "u_CAD4S001", 0, 2, , .F.})

    RestArea(aArea)
Return
```

| Posição do array | Valor | Significado |
|---|---|---|
| 1 | `"Impressão d4Sign"` | Título do botão |
| 2 | `"u_CAD4S001"` | Função chamada |
| 3 | `0` | Nível de acesso |
| 4 | `2` | Tipo de operação (Impressão) |
| 6 | `.F.` | Botão visível (não oculto) |

### CAD4S000 — Classe D4SignClient

**Arquivo:** `CAD4S000.tlpp`

Implementa a classe `D4SignClient`, responsável por toda a comunicação HTTP com a API REST do D4Sign: autenticação, requisições multipart, upload de documentos, gerenciamento de signatários e download dos arquivos assinados.

#### Atributos

| Atributo | Descrição |
|---|---|
| `cToken` | Token de autenticação da API D4Sign (live ou sandbox). |
| `cCrypt` | Chave de criptografia (cryptKey) complementar ao token. |
| `cBaseURL` | URL base da API (`https://secure.d4sign.com.br/api/v1` em produção). |
| `cCofreId` | UUID do cofre onde os documentos são armazenados. |
| `cPastaID` | UUID da pasta dentro do cofre (reservado). |
| `cPastDoc` | Caminho do documento (reservado). |
| `StatusCode` | Código HTTP da última requisição. |
| `Result` | Corpo da resposta da última requisição. |

#### Métodos públicos

| Método | Parâmetros | Descrição |
|---|---|---|
| `New()` | — | Construtor. Detecta o ambiente (produção/sandbox) e carrega tokens/URLs via `SuperGetMV`. |
| `Requisicao()` | `cEndpoint, cMetodo, aHeader, cBody` | Método central de comunicação. Executa GET, POST ou DELETE via `FWRest`. |
| `UploadDocument()` | `cFilePath, cNomeDoc` | Envia um PDF ao cofre D4Sign via `multipart/form-data` com Base64. |
| `AddSigner()` | `cUUIDDoc, cNome, cEmail, cCpf` | Cadastra o colaborador como signatário do documento. |
| `SendToSign()` | `cUUIDDoc` | Dispara os e-mails de solicitação de assinatura. |
| `GetStatus()` | `cUUIDDoc` | Consulta o status atual do documento (`GET /documents/{uuid}`). |
| `DownloadSigned()` | `cUUIDDoc, cSavePath` | Solicita o link de download do documento assinado. |
| `GetResult()` | — | Retorna o corpo da última resposta da API. |
| `GetStatusCode()` | — | Retorna o código HTTP da última requisição. |

#### Detecção de ambiente

O método `New()` detecta o ambiente pelo nome do servidor (`GetenvServer`). Se o nome contiver a string `CAST`, utiliza os parâmetros de produção (`MV_`); caso contrário, usa o ambiente **sandbox** com valores fixos no código.

#### Upload multipart

O `UploadDocument` converte o PDF para Base64 (`Encode64`) e monta o corpo `multipart/form-data` com os campos:

- `base64_binary_file` — conteúdo do PDF em Base64
- `mime_type` — fixo `application/pdf`
- `name` — nome do arquivo (usado também para determinar a pasta via `fGetFolder`)
- `uuid_folder` — UUID da pasta D4Sign, obtido via parâmetro `CS_D4FOLxx`

#### Configuração fixa do signatário (AddSigner)

| Campo | Valor | Significado |
|---|---|---|
| `act` | `'1'` | Ação: Assinar |
| `foreign` | `'0'` | Possui CPF brasileiro |
| `certificadoicpbr` | `'0'` | Assinatura padrão D4Sign (não ICP-Brasil) |
| `assinatura_presencial` | `'0'` | Assinatura remota |

### CAD4S001 — Geração do Recibo em PDF

**Arquivo:** `CAD4S001.PRX` · **Includes:** `U_CAD4S001.ch` / `U_CAD4S001_pt-br.tres`

Reescrita da rotina padrão de impressão de férias do Protheus (GP130), adaptada para o componente **`FWMsPrinter`**, que gera PDFs diretamente no servidor, sem impressora física.

A função `u_CAD4S001` (chamada pelo botão do PE):

1. Verifica restrições de acesso a dados sensíveis (`ChkOfusca` / `FwProtectedDataUtil`);
2. Exibe o diálogo de parâmetros de impressão (pergunte **GPEM030**);
3. Chama a função estática `GP130Imp` para processar os registros e gerar os PDFs em `<CS_D4SDIR>\pendente\`.

#### Parâmetros de impressão (GPEM030)

| Parâmetro | Descrição |
|---|---|
| `mv_par01` | Solicitar 1ª parcela do 13º salário |
| `mv_par02` | Solicitar Abono Pecuniário |
| `mv_par03` | Imprimir Aviso de Férias |
| `mv_par04` | Imprimir Recibo de Férias |
| `mv_par05` | Imprimir Recibo de Abono |
| `mv_par06` | Imprimir Recibo da 1ª parcela do 13º |
| `mv_par07` | Imprimir período de férias |
| `mv_par08/09` | Período de férias (De / Até) |
| `mv_par10–17` | Filtros: Filial, Matrícula, Centro de Custo, Nome (De / Até) |
| `mv_par18` | Data de solicitação do 13º |
| `mv_par19` | Número de vias a imprimir |
| `mv_par20/21` | Data de pagamento (De / Até) |

#### Processamento (GP130Imp)

Percorre a tabela `SRA` (Funcionários) respeitando os filtros e, para cada funcionário com período de férias vigente:

- Carrega o período de cálculo de férias (`fGetLastPer`, `fCarPeriodo`);
- Carrega as tabelas de apuração de dias de férias, considerando regime parcial (Art. 130-A da CLT);
- Consulta o arquivo `SRF` para localizar o período de férias programado;
- Calcula dias de férias, abono pecuniário, deduções por faltas e demais fatores;
- Chama `ImpFerpdf()` (impressão via `FWMsPrinter`) para cada via solicitada.

A função auxiliar `fLogoEmp()` localiza e copia o logotipo da empresa para uma pasta temporária do servidor, buscando o arquivo na ordem: **Grupo+Empresa+Unidade+Filial → Empresa+Filial → `LGRL01.BMP` (padrão)**.

#### Internacionalização (i18n)

Os textos dos recibos (aviso de férias, solicitação de abono, 1ª parcela do 13º etc.) são definidos via `FWI18NLang` no include `U_CAD4S001.ch` e no arquivo de recurso `U_CAD4S001_pt-br.tres`, permitindo tradução sem alteração do fonte.

### CAD4S003 — Envio para Assinatura

**Arquivo:** `CAD4S003.tlpp` · **Execução:** Job agendado (Schedule)

Varre a pasta de PDFs pendentes, faz o upload de cada documento, cadastra o colaborador como signatário e dispara a assinatura.

| # | Etapa | Detalhe |
|---|---|---|
| 1 | Varredura | Lista todos os `.pdf` em `<CS_D4SDIR>\pendente\`. |
| 2 | Extração do nome | Interpreta o nome do arquivo (empresa, filial, matrícula, data — ver [convenção](#convenção-de-nome-dos-arquivos-pdf)). |
| 3 | Upload | `oD4Sign:UploadDocument()` → obtém o UUID do documento no D4Sign. |
| 4 | Consulta SRA | Busca `RA_NOME`, `RA_EMAIL`, `RA_XEMAIL2`, `RA_CIC` usando empresa/filial/matrícula. |
| 5 | AddSigner | Cadastra o colaborador como signatário (e-mail principal `RA_EMAIL`). |
| 6 | SendToSign | Dispara o e-mail de assinatura. |
| 7 | Atualiza SRH | `UPDATE SRH`: grava `RH_IDD4SIG` (UUID) e `RH_D4SIGN = '2'`. |
| 8 | Move arquivo | Copia o PDF de `pendente\` para `enviado\<empresa>\` e apaga o original. |
| 9 | Erro | Falha em etapa HTTP → `RH_D4SIGN = '5'` e retorno da API gravado em `RH_IDD4SIG`. |

### CAD4S004 — Download dos Assinados

**Arquivo:** `CAD4S004.tlpp` · **Execução:** Job agendado (Schedule)

Monitora os documentos enviados e baixa os já assinados pelo colaborador.

| # | Etapa | Detalhe |
|---|---|---|
| 1 | Consulta SRH | Query `UNION` para todas as empresas (via `U_GetEmpRA`), buscando `RH_D4SIGN IN ('1','2')`. |
| 2 | DownloadSigned | `oD4Sign:DownloadSigned()` com o UUID de `RH_IDD4SIG` → API retorna JSON com `name` e `url` (link temporário do ZIP). |
| 3 | Download do ZIP | `HttpGet(cUrl)` e gravação em disco via `MemoWrite`. |
| 4 | Extração | Descompacta o ZIP em `<CS_D4SDIR>\assinado\<empresa>\` usando `fUnzip`. |
| 5 | Limpeza | Remove o ZIP, a pasta `originais` e conteúdos não assinados do pacote D4Sign. |
| 6 | Atualiza SRH | `RH_D4SIGN = '4'` (Baixado / Concluído). |
| 7 | Erro | HTTP fora de 200–299 → `RH_D4SIGN = '5'` e registro do retorno da API. |

### CAD4S005 — Reenvio de Documentos com Erro

**Arquivo:** `CAD4S005.tlpp` · **Execução:** Job agendado (Schedule)

Job de contingência que reprocessa documentos que falharam na etapa de cadastro de signatário ou de envio para assinatura (`RH_D4SIGN = '5'`). Composto por duas rotinas:

**`ReenviaSignatario()`**
- Varre `<CS_D4SDIR>\erro\addsigner\` em busca de PDFs;
- Extrai empresa/filial/matrícula/data do nome do arquivo;
- Consulta `SRA` × `SRH` (JOIN por filial, matrícula e `RH_DTRECIB`) filtrando `RH_D4SIGN = '5'`;
- Reexecuta `AddSigner` e, em caso de sucesso, `SendToSign`;
- Sucesso completo → `RH_D4SIGN = '2'` e move o PDF para `enviado\<empresa>\`;
- Falha no `SendToSign` → move o PDF para `erro\sendsigner\` (reprocessado pela rotina seguinte).

**`ReenviaAssinatura()`**
- Varre `<CS_D4SDIR>\erro\sendsigner\`;
- Mesma lógica de consulta, reexecutando apenas o `SendToSign` (o signatário já foi cadastrado);
- Sucesso → `RH_D4SIGN = '2'` e move o PDF para `enviado\<empresa>\`.

---

## Convenção de Nome dos Arquivos PDF

O nome do arquivo gerado pelo CAD4S001 segue estrutura **posicional** — o CAD4S003 e o CAD4S005 extraem os dados diretamente da posição dos caracteres:

```
EEFF_MMMMMM_AAAAMMDD.pdf
```

| Posição | Conteúdo |
|---|---|
| 1–2 | Empresa (`EE`) |
| 3–4 | Filial (`FF`) |
| 5 | `_` |
| 6–11 | Matrícula (`MMMMMM`) |
| 12 | `_` |
| 13–20 | Data do recibo (`AAAAMMDD`) |

**Exemplo:** `0101_000001_20260401.pdf` → Empresa 01, Filial 01, Matrícula 000001, recibo de 01/04/2026.

> ⚠️ Qualquer alteração no padrão de nomenclatura exige ajuste correspondente nas funções `SubStr` do CAD4S003 e do CAD4S005.

## Estrutura de Diretórios

Diretório raiz definido pelo parâmetro `CS_D4SDIR`:

```
<CS_D4SDIR>\
├── pendente\              → PDFs gerados pelo CAD4S001, aguardando envio
├── enviado\<empresa>\     → PDFs após upload bem-sucedido
├── assinado\<empresa>\    → PDFs assinados baixados pelo CAD4S004
└── erro\
    ├── addsigner\         → falha no cadastro do signatário (reprocessa CAD4S005)
    └── sendsigner\        → falha no envio para assinatura (reprocessa CAD4S005)
```

## Parâmetros do Sistema (MV_)

| Parâmetro | Descrição |
|---|---|
| `CS_D4TOKEN` | Token de autenticação da API D4Sign (`live_...`). |
| `CS_D4CRYPT` | Chave de criptografia (cryptKey) da API. |
| `CS_D4COFRE` | UUID do cofre principal no D4Sign. |
| `CS_D4FOLxx` | UUID da pasta D4Sign por empresa — o sufixo `xx` corresponde aos 2 primeiros caracteres do nome do documento (ex.: `CS_D4FOL01` para a empresa `01`). |
| `CS_D4SDIR` | Diretório raiz dos PDFs de férias (padrão: `C:\d4sign\rh\ferias\docs\`). |

> 🔒 Os valores dos tokens **não** devem ser versionados neste repositório. Cadastre-os apenas no configurador (SX6) do ambiente.

## Ciclo de Vida do Documento (RH_D4SIGN)

O campo `RH_D4SIGN` na tabela `SRH` controla o ciclo de vida do documento na integração:

| Código | Status | Descrição |
|---|---|---|
| `'1'` | Enviado | Documento enviado ao D4Sign, aguardando confirmação. |
| `'2'` | Aguardando Assinatura | Signatário cadastrado e e-mail disparado. |
| `'3'` | Assinado | Documento assinado (verificação via `GetStatus`). |
| `'4'` | Baixado / Concluído | Download do assinado realizado. Processo encerrado. |
| `'5'` | Erro na Integração | Falha em alguma etapa. Detalhes em `RH_IDD4SIG`. |

## Dicionário de Dados — Campos Customizados

A tabela `SRH` (Férias) deve conter os campos:

| Campo | Tipo | Descrição |
|---|---|---|
| `RH_IDD4SIG` | Caractere | UUID do documento no D4Sign ou mensagem de erro da API (quando status `'5'`). |
| `RH_D4SIGN` | Caractere (1) | Status da integração (`'1'` a `'5'`). |

## Instalação e Configuração

1. **Compilar os fontes** no repositório do ambiente: `CAD4S000.tlpp`, `CAD4S001.PRX` (+ `U_CAD4S001.ch` e `U_CAD4S001_pt-br.tres`), `CAD4S003.tlpp`, `CAD4S004.tlpp`, `CAD4S005.tlpp` e `PE_GP030MNU.tlpp`.
2. **Criar os campos customizados** `RH_IDD4SIG` e `RH_D4SIGN` na tabela `SRH` (SX3).
3. **Cadastrar os parâmetros** `CS_D4TOKEN`, `CS_D4CRYPT`, `CS_D4COFRE`, `CS_D4FOLxx` e `CS_D4SDIR` no configurador (SX6).
4. **Criar a estrutura de diretórios** no servidor conforme [Estrutura de Diretórios](#estrutura-de-diretórios).
5. **Agendar os jobs** no Schedule Manager do Protheus:
   - `U_CAD4S003` — recomendado a cada **15–30 minutos**;
   - `U_CAD4S004` — recomendado a cada **1 hora** (ou conforme SLA de assinatura do RH);
   - `U_CAD4S005` — conforme necessidade de reprocessamento de erros.
6. Validar o botão **"Impressão d4Sign"** na rotina GP130 (módulo SIGAGPE).

## Considerações Técnicas e Boas Práticas

**Segurança de credenciais**
- Tokens (`CS_D4TOKEN`, `CS_D4CRYPT`) devem ficar exclusivamente em parâmetros de sistema — nunca em código-fonte de produção nem no repositório.
- O ambiente sandbox possui credenciais fixas no código: remover ou proteger antes de qualquer deploy em produção.
- Avaliar criptografia adicional dos parâmetros sensíveis via `FWProtectedData`.

**Tratamento de erros**
- Registros com `RH_D4SIGN = '5'` devem ser monitorados; o campo `RH_IDD4SIG` contém o erro retornado pela API para diagnóstico.
- O CAD4S005 automatiza o reprocessamento dos erros de `AddSigner` e `SendToSign`.
- Recomenda-se implementar alertas automáticos (e-mail/notificação) para documentos com status de erro.

**Cadastro de colaboradores**
- O e-mail do signatário é o campo `RA_EMAIL` da tabela `SRA`; o campo `RA_XEMAIL2` também é recuperado (disponível para cenários com múltiplos e-mails).
- Garantir que os e-mails estejam cadastrados e válidos antes de executar o CAD4S003.

## Referências

| Recurso | Detalhe |
|---|---|
| D4Sign API v1 (produção) | `https://secure.d4sign.com.br/api/v1` |
| D4Sign API v1 (sandbox) | `https://sandbox.d4sign.com.br/api/v1` |
| Rotina Protheus | GP130 — Impressão de Recibos de Férias (SIGAGPE) |
| Tabelas envolvidas | `SRA` (Funcionários), `SRH` (Férias), `SRF` (Programação de Férias) |
| Autor | Silvano Franca — ALWA |
