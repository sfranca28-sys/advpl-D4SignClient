# Referência Técnica — Classe `D4SignClient` e Funções da Integração

Documentação de referência da classe, métodos e funções da integração **Protheus × D4Sign** (assinatura digital de recibos de férias), com exemplos de uso extraídos do código real.

> **Linguagem:** TLPP / AdvPL · **Versão Protheus:** 12.2310+ · **Autor:** Silvano Franca — ALWA

---

## Índice

- [Classe D4SignClient (CAD4S000)](#classe-d4signclient-cad4s000)
  - [Atributos](#atributos)
  - [New()](#new--construtor)
  - [Requisicao()](#requisicaocendpoint-cmetodo-aheader-cbody)
  - [UploadDocument()](#uploaddocumentcfilepath-cnomedoc)
  - [AddSigner()](#addsignercuuiddoc-cnome-cemail-ccpf)
  - [SendToSign()](#sendtosigncuuiddoc)
  - [GetStatus()](#getstatuscuuiddoc)
  - [DownloadSigned()](#downloadsignedcuuiddoc-csavepath)
  - [GetResult() / GetStatusCode()](#getresult--getstatuscode)
  - [Funções estáticas internas](#funções-estáticas-internas)
- [Funções de Usuário (User Functions)](#funções-de-usuário-user-functions)
- [Exemplo completo — Fluxo de ponta a ponta](#exemplo-completo--fluxo-de-ponta-a-ponta)
- [Padrão de tratamento de erro](#padrão-de-tratamento-de-erro)
- [Observações de implementação](#observações-de-implementação)

---

## Classe `D4SignClient` (CAD4S000)

**Arquivo:** `CAD4S000.tlpp`

Encapsula toda a comunicação HTTP com a API REST do D4Sign (`FWRest`): autenticação, upload multipart, gerenciamento de signatários, consulta de status e download de documentos assinados.

```xbase
#Include "TOTVS.ch"
#Include "Tlpp-core.th"

Class D4SignClient

    Data cToken     as character
    Data cCrypt     as character
    Data cBaseURL   as character
    Data cCofreId   as character
    Data cPastaID   as character
    Data cPastDoc   as character
    Data StatusCode as numeric
    Data Result     as character

    Public Method New()
    Public Method Requisicao(cEndpoint, cMetodo, aHeader, cBody)
    Public Method UploadDocument(cFilePath, cNomeDoc)   // Enviar PDF para o cofre
    Public Method AddSigner(cUUIDDoc, cNome, cEmail)    // Registrar signatário
    Public Method SendToSign(cUUIDDoc)                  // Disparar e-mails de assinatura
    Public Method GetStatus(cUUIDDoc)                   // Monitorar status do documento
    Public Method DownloadSigned(cUUIDDoc, cSavePath)   // Baixar documento assinado
    Public Method GetResult()
    Public Method GetStatusCode()

EndClass
```

### Atributos

| Atributo | Tipo | Descrição |
|---|---|---|
| `cToken` | character | Token de autenticação da API D4Sign (`tokenAPI`). |
| `cCrypt` | character | Chave de criptografia (`cryptKey`) complementar ao token. |
| `cBaseURL` | character | URL base da API. Produção: `https://secure.d4sign.com.br/api/v1` · Sandbox: `https://sandbox.d4sign.com.br/api/v1`. |
| `cCofreId` | character | UUID do cofre D4Sign onde os documentos são armazenados. |
| `cPastaID` | character | UUID da pasta dentro do cofre *(reservado — não utilizado atualmente)*. |
| `cPastDoc` | character | Caminho do documento *(reservado)*. |
| `StatusCode` | numeric | Código HTTP retornado pela **última** requisição executada. |
| `Result` | character | Corpo (JSON) da resposta da **última** requisição, ou mensagem de erro do `FWRest`. |

> A classe é **stateful por requisição**: cada chamada de método sobrescreve `StatusCode` e `Result`. Sempre leia `GetStatusCode()` / `GetResult()` **imediatamente** após cada chamada, antes de executar a próxima.

---

### `New()` — Construtor

Instancia o cliente, prepara o ambiente e carrega as credenciais.

**Sintaxe**

```xbase
oD4Sign := D4SignClient():New()
```

**Retorno:** `Self` (a própria instância).

**Comportamento**

1. **Preparação de ambiente para jobs:** se não houver ambiente aberto (`Select("SX2") == 0`), executa `RpcSetType(3)` e `RpcSetEnv("01","01")` — isso permite que a classe seja usada em jobs agendados sem preparação prévia.
2. **Detecção de ambiente:** verifica o nome do servidor via `GetenvServer()`:
   - Se contiver `CASTWFL` ou `CASTCOMP` → **produção**: carrega `CS_D4TOKEN`, `CS_D4CRYPT` e `CS_D4COFRE` via `SuperGetMV` e aponta para `secure.d4sign.com.br`;
   - Caso contrário → **sandbox**: usa credenciais fixas no código e aponta para `sandbox.d4sign.com.br`.

**Exemplo**

```xbase
Local oD4Sign as object

oD4Sign := D4SignClient():New()

ConOut("Ambiente: " + oD4Sign:cBaseURL)
ConOut("Cofre:    " + oD4Sign:cCofreId)
```

---

### `Requisicao(cEndpoint, cMetodo, aHeader, cBody)`

Método central de comunicação HTTP. Todos os demais métodos da classe (exceto `UploadDocument`, que monta a requisição manualmente) delegam para ele.

**Parâmetros**

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `cEndpoint` | character | Sim | Endpoint relativo à `cBaseURL` (ex.: `/documents/{uuid}`). |
| `cMetodo` | character | Sim | Verbo HTTP: `"POST"`, `"GET"` ou `"DELETE"`. |
| `aHeader` | array | Não | Array de strings de headers HTTP (ex.: `{"content-type: application/json"}`). |
| `cBody` | character | Não | Corpo da requisição (JSON) para POST; query string para GET/DELETE. |

**Retorno:** nenhum (resultado disponível em `GetStatusCode()` / `GetResult()`).

**Autenticação:** o token e a cryptKey são enviados automaticamente na **query string** de todas as chamadas:

```
{endpoint}?tokenAPI={cToken}&cryptKey={cCrypt}
```

**Exemplo — chamada genérica à API**

```xbase
Local oD4Sign := D4SignClient():New()
Local aHeader := {}

aAdd(aHeader, "accept: application/json")

// Lista os cofres disponíveis na conta
oD4Sign:Requisicao("/safes", "GET", aHeader, "")

If oD4Sign:GetStatusCode() >= 200 .And. oD4Sign:GetStatusCode() < 300
    ConOut(oD4Sign:GetResult())   // JSON com os cofres
Else
    ConOut("Erro HTTP " + cValToChar(oD4Sign:GetStatusCode()))
EndIf
```

---

### `UploadDocument(cFilePath, cNomeDoc)`

Envia um arquivo PDF para o cofre configurado no D4Sign, via `POST /documents/{cofre}/uploadbinary` com corpo `multipart/form-data`.

**Parâmetros**

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `cFilePath` | character | Caminho completo do PDF no servidor (ex.: `C:\d4sign\rh\ferias\docs\pendente\0101_000001_20260401.pdf`). |
| `cNomeDoc` | character | Nome do documento no D4Sign. **Importante:** os 2 primeiros caracteres determinam a pasta de destino (via parâmetro `CS_D4FOLxx` — ver [`fGetFolder`](#funções-estáticas-internas)). |

**Retorno:** nenhum. Em caso de sucesso, `GetResult()` retorna um JSON contendo o **`uuid`** do documento criado — guarde-o, pois ele é a chave para todas as operações seguintes.

**Campos do multipart enviado**

| Campo | Conteúdo |
|---|---|
| `base64_binary_file` | PDF convertido para Base64 (`Encode64`). |
| `mime_type` | `application/pdf` (fixo). |
| `name` | `cNomeDoc`. |
| `uuid_folder` | UUID da pasta, resolvido por `fGetFolder(Left(cNomeDoc,2))`. |

**Exemplo**

```xbase
Local oD4Sign  := D4SignClient():New()
Local jRetorno := JsonObject():New()
Local cUUIDDoc := ""
Local cArquivo := "C:\d4sign\rh\ferias\docs\pendente\0101_000001_20260401.pdf"
Local cNomeDoc := "0101_000001_20260401.pdf"

oD4Sign:UploadDocument(cArquivo, cNomeDoc)

If oD4Sign:GetStatusCode() >= 200 .And. oD4Sign:GetStatusCode() < 300
    jRetorno:fromJson(oD4Sign:GetResult())
    If jRetorno["uuid"] != Nil
        cUUIDDoc := jRetorno["uuid"]
        ConOut("Documento criado no D4Sign: " + cUUIDDoc)
    EndIf
Else
    ConOut("Falha no upload: " + oD4Sign:GetResult())
EndIf
```

---

### `AddSigner(cUUIDDoc, cNome, cEmail, cCpf)`

Cadastra o colaborador como signatário do documento, via `POST /documents/{uuid}/createlist`.

**Parâmetros**

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `cUUIDDoc` | character | UUID do documento (retornado pelo `UploadDocument`). |
| `cNome` | character | Nome do signatário (obtido de `SRA->RA_NOME`). |
| `cEmail` | character | E-mail do signatário (obtido de `SRA->RA_EMAIL`) — **é a chave de identificação no D4Sign**. |
| `cCpf` | character | CPF do signatário (obtido de `SRA->RA_CIC`). |

> ⚠️ Na implementação atual, apenas o **e-mail** é efetivamente enviado no JSON de signatários; nome e CPF são recebidos mas não incluídos no corpo (ver [Observações de implementação](#observações-de-implementação)).

**Configurações fixas do signatário**

| Campo JSON | Valor | Significado |
|---|---|---|
| `act` | `"1"` | Ação: **Assinar** (a API suporta 1–13: aprovar, testemunha, fiador etc.). |
| `foreign` | `"0"` | Possui CPF brasileiro. |
| `certificadoicpbr` | `"0"` | Assinatura padrão D4Sign (não ICP-Brasil). |
| `assinatura_presencial` | `"0"` | Assinatura remota. |

**Corpo enviado (exemplo)**

```json
{
  "signers": [
    {
      "email": "colaborador@empresa.com.br",
      "act": "1",
      "foreign": "0",
      "certificadoicpbr": "0",
      "assinatura_presencial": "0"
    }
  ]
}
```

**Exemplo — consultando o colaborador na SRA**

```xbase
Local oD4Sign := D4SignClient():New()
Local cQuery  := ""

cQuery := " SELECT RA_NOME, RA_EMAIL, RA_CIC "
cQuery += " FROM SRA010 "
cQuery += " WHERE D_E_L_E_T_ = '' AND "
cQuery += "       RA_FILIAL = '01' AND "
cQuery += "       RA_MAT = '000001' "

MpSysOpenQuery(cQuery, "tSRA")

If tSRA->( !EOF() )
    oD4Sign:AddSigner( cUUIDDoc            ,;
                       AllTrim(tSRA->RA_NOME) ,;
                       AllTrim(tSRA->RA_EMAIL),;
                       AllTrim(tSRA->RA_CIC)  )

    If oD4Sign:GetStatusCode() >= 200 .And. oD4Sign:GetStatusCode() < 300
        ConOut("Signatário cadastrado com sucesso")
    EndIf
EndIf
tSRA->( DbCloseArea() )
```

---

### `SendToSign(cUUIDDoc)`

Dispara os e-mails de solicitação de assinatura para os signatários já cadastrados no documento, via `POST /documents/{uuid}/sendtosigner`.

**Parâmetros**

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `cUUIDDoc` | character | UUID do documento no D4Sign. |

**Retorno:** nenhum (verificar `GetStatusCode()`).

**Exemplo**

```xbase
oD4Sign:SendToSign(cUUIDDoc)

If oD4Sign:GetStatusCode() >= 200 .And. oD4Sign:GetStatusCode() < 300
    // Atualiza status no Protheus: 2 = Aguardando Assinatura
    TcSqlExec("UPDATE SRH010 SET RH_D4SIGN = '2' WHERE ... ")
EndIf
```

---

### `GetStatus(cUUIDDoc)`

Consulta o status atual do documento, via `GET /documents/{uuid}`.

**Parâmetros**

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `cUUIDDoc` | character | UUID do documento no D4Sign. |

**Retorno:** nenhum. `GetResult()` traz um **array JSON** — como a consulta é feita por um único UUID, o documento está sempre na **posição 1**.

**Principais status do D4Sign (`statusId`)**

| statusId | Significado |
|---|---|
| `"1"` | Em processamento |
| `"2"` | Aguardando signatários |
| `"3"` | Aguardando assinaturas |
| `"4"` | **Finalizado (assinado)** — pronto para download |
| `"5"` | Arquivado |
| `"6"` | Cancelado |

**Exemplo — verificando se o documento foi assinado**

```xbase
Local jRetorno := JsonObject():New()
Local cStatus  := ""

oD4Sign:GetStatus(cUUIDDoc)

If oD4Sign:GetStatusCode() >= 200 .And. oD4Sign:GetStatusCode() < 300
    jRetorno:fromJson(oD4Sign:GetResult())
    cStatus := jRetorno[1]["statusId"]   // retorno é array: posição 1

    If cStatus == "4"
        ConOut("Documento assinado! Pronto para download.")
    EndIf
EndIf
```

---

### `DownloadSigned(cUUIDDoc, cSavePath)`

Solicita o link de download do documento assinado, via `POST /documents/{uuid}/download`.

**Parâmetros**

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `cUUIDDoc` | character | UUID do documento no D4Sign. |
| `cSavePath` | character | Diretório de destino no servidor *(informativo — o download em si é feito pelo chamador; ver exemplo)*. |

**Corpo enviado (fixo)**

```json
{
  "type": "pdf",
  "language": "pt",
  "encoding": false,
  "document": false,
  "videoselfie_frame": false
}
```

**Retorno:** nenhum. `GetResult()` traz um JSON com:

| Campo | Descrição |
|---|---|
| `name` | Nome do arquivo. |
| `url` | **URL temporária** de um ZIP contendo o PDF assinado. |

> A URL é temporária — o download (`HttpGet`) deve ser feito imediatamente após a chamada.

**Exemplo — download, extração e limpeza (padrão do CAD4S004)**

```xbase
Local jRetorno := JsonObject():New()
Local cDirDow  := "C:\d4sign\rh\ferias\docs\assinado\01\"
Local cArquivo := ""
Local cUrl     := ""
Local cArqZip  := ""
Local nRetorno := 0

oD4Sign:DownloadSigned(cUUIDDoc, cDirDow)

If oD4Sign:GetStatusCode() >= 200 .And. oD4Sign:GetStatusCode() < 300
    jRetorno:fromJson(oD4Sign:GetResult())
    cArquivo := jRetorno["name"]
    cUrl     := jRetorno["url"]
    cArqZip  := cDirDow + cArquivo + ".zip"

    // Baixa o ZIP pela URL temporária e grava em disco
    If MemoWrite(cArqZip, HttpGet(cUrl))
        nRetorno := fUnzip(cArqZip, cDirDow)
        If nRetorno == 0
            fErase(cArqZip)   // remove o ZIP após extração
            // ... remover pasta "originais\" com os PDFs não assinados do pacote
        EndIf
    EndIf
EndIf
```

---

### `GetResult()` / `GetStatusCode()`

Acessores do resultado da última requisição.

| Método | Retorno | Descrição |
|---|---|---|
| `GetResult()` | character | Corpo (JSON) da última resposta da API, ou mensagem de erro do `FWRest`. |
| `GetStatusCode()` | numeric | Código HTTP da última requisição (ex.: `200`, `401`, `500`). |

**Exemplo — padrão de verificação**

```xbase
nStatus := oD4Sign:GetStatusCode()

If nStatus >= 200 .And. nStatus < 300
    // Sucesso — processar oD4Sign:GetResult()
Else
    // Falha — logar/gravar oD4Sign:GetResult() para diagnóstico
EndIf
```

---

### Funções estáticas internas

Funções auxiliares privadas do `CAD4S000.tlpp` (não acessíveis fora do fonte):

#### `D4S_BuildMultipart(cBoundary, aParts)`

Monta o corpo `multipart/form-data` da requisição de upload.

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `cBoundary` | character | Delimitador do multipart (gerado dinamicamente: `----ProtheusBoundary` + hora). |
| `aParts` | array | Array de pares `{nome_do_campo, valor}`. |

**Retorno:** `character` — corpo multipart completo.

```xbase
// Uso interno no UploadDocument:
aAdd(aParts, {"base64_binary_file", cBase64})
aAdd(aParts, {"mime_type", "application/pdf"})
aAdd(aParts, {"name", cNomeDoc})
aAdd(aParts, {"uuid_folder", cFolder})

cBody := D4S_BuildMultipart(cBoundary, aParts)
```

#### `fGetFolder(xEmpresa)`

Resolve o UUID da pasta D4Sign a partir do código da empresa.

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `xEmpresa` | character | Código da empresa (2 caracteres — extraído do início do nome do documento). |

**Retorno:** `character` — UUID da pasta, lido do parâmetro `CS_D4FOL{xEmpresa}` (ex.: empresa `01` → parâmetro `CS_D4FOL01`).

---

## Funções de Usuário (User Functions)

### `U_GP030MNU()` — Ponto de entrada

**Arquivo:** `PE_GP030MNU.tlpp` · **Contexto:** executado automaticamente pelo Protheus ao abrir o menu da rotina GP130.

Adiciona o botão **"Impressão d4Sign"** à barra de ações, vinculado a `u_CAD4S001`.

```xbase
User Function GP030MNU()
    Local aArea := GetArea()

    // Adiciona o botão de impressão para a nova rotina de férias
    AAdd(ARotina, { "Impressão d4Sign", "u_CAD4S001", 0, 2, , .F.})

    RestArea(aArea)
Return
```

### `U_CAD4S001()` — Geração do recibo em PDF

**Arquivo:** `CAD4S001.PRX` · **Contexto:** interativo (chamado pelo botão na GP130).

Exibe o diálogo de parâmetros (pergunte `GPEM030`), processa os funcionários da `SRA` com férias vigentes e gera os PDFs via `FWMsPrinter` em `<CS_D4SDIR>\pendente\`. Não recebe parâmetros — toda a parametrização vem do pergunte.

### `U_CAD4S003()` — Job de envio para assinatura

**Arquivo:** `CAD4S003.tlpp` · **Contexto:** job agendado (Schedule).

Para cada PDF em `pendente\`: faz upload, cadastra o signatário, dispara a assinatura, atualiza a `SRH` e move o arquivo. Não recebe parâmetros.

```xbase
// Agendamento via Schedule ou execução manual:
U_CAD4S003()
```

### `U_CAD4S004()` — Job de monitoramento e download

**Arquivo:** `CAD4S004.tlpp` · **Contexto:** job agendado (Schedule).

Consulta a `SRH` de todas as empresas (`RH_D4SIGN IN ('1','2','5')`), verifica o status de cada documento via `GetStatus()` e, quando `statusId == "4"` (assinado), baixa e extrai o PDF em `assinado\<empresa>\`, atualizando `RH_D4SIGN = '4'`.

```xbase
U_CAD4S004()
```

### `U_CAD4S005()` — Job de reenvio de erros

**Arquivo:** `CAD4S005.tlpp` · **Contexto:** job agendado (Schedule).

Reprocessa documentos com falha (`RH_D4SIGN = '5'`):

- `ReenviaSignatario()` — varre `erro\addsigner\`, reexecuta `AddSigner` + `SendToSign`;
- `ReenviaAssinatura()` — varre `erro\sendsigner\`, reexecuta apenas `SendToSign`.

```xbase
U_CAD4S005()
```

---

## Exemplo completo — Fluxo de ponta a ponta

Exemplo consolidado do ciclo upload → signatário → assinatura → monitoramento → download, seguindo o padrão real dos jobs CAD4S003/CAD4S004:

```xbase
#Include "totvs.ch"
#Include "Tlpp-core.th"

User Function ExemploD4S()

    Local oD4Sign  := D4SignClient():New()
    Local jRetorno := JsonObject():New()
    Local cArquivo := "C:\d4sign\rh\ferias\docs\pendente\0101_000001_20260401.pdf"
    Local cNomeDoc := "0101_000001_20260401.pdf"
    Local cUUIDDoc := ""
    Local nStatus  := 0
    Local cStatus  := ""
    Local cDirDow := "C:\d4sign\rh\ferias\docs\assinado\01\"
    Local cUrl    := ""
    Local cArqZip := ""

    // ------------------------------------------------------------------
    // 1) UPLOAD do PDF para o cofre
    // ------------------------------------------------------------------
    oD4Sign:UploadDocument(cArquivo, cNomeDoc)
    nStatus := oD4Sign:GetStatusCode()

    If !(nStatus >= 200 .And. nStatus < 300)
        ConOut("[D4Sign] Erro no upload: " + oD4Sign:GetResult())
        Return
    EndIf

    jRetorno:fromJson(oD4Sign:GetResult())
    cUUIDDoc := jRetorno["uuid"]

    // ------------------------------------------------------------------
    // 2) ADDSIGNER — cadastra o colaborador como signatário
    // ------------------------------------------------------------------
    oD4Sign:AddSigner(cUUIDDoc, "JOSE DA SILVA", "jose.silva@empresa.com.br", "12345678900")
    nStatus := oD4Sign:GetStatusCode()

    If !(nStatus >= 200 .And. nStatus < 300)
        ConOut("[D4Sign] Erro ao cadastrar signatário: " + oD4Sign:GetResult())
        Return
    EndIf

    // ------------------------------------------------------------------
    // 3) SENDTOSIGN — dispara o e-mail de assinatura
    // ------------------------------------------------------------------
    oD4Sign:SendToSign(cUUIDDoc)
    nStatus := oD4Sign:GetStatusCode()

    If !(nStatus >= 200 .And. nStatus < 300)
        ConOut("[D4Sign] Erro ao enviar para assinatura: " + oD4Sign:GetResult())
        Return
    EndIf

    ConOut("[D4Sign] Documento " + cUUIDDoc + " aguardando assinatura")
    // Neste ponto, gravar RH_IDD4SIG = cUUIDDoc e RH_D4SIGN = '2' na SRH

    // ------------------------------------------------------------------
    // 4) GETSTATUS — monitoramento (normalmente em job separado)
    // ------------------------------------------------------------------
    oD4Sign:GetStatus(cUUIDDoc)
    nStatus := oD4Sign:GetStatusCode()

    If nStatus >= 200 .And. nStatus < 300
        jRetorno:fromJson(oD4Sign:GetResult())
        cStatus := jRetorno[1]["statusId"]   // resposta é um array

        // ----------------------------------------------------------
        // 5) DOWNLOADSIGNED — se assinado, baixa e extrai
        // ----------------------------------------------------------
        If cStatus == "4"

            oD4Sign:DownloadSigned(cUUIDDoc, cDirDow)
            nStatus := oD4Sign:GetStatusCode()

            If nStatus >= 200 .And. nStatus < 300
                jRetorno:fromJson(oD4Sign:GetResult())
                cArqZip := cDirDow + jRetorno["name"] + ".zip"
                cUrl    := jRetorno["url"]

                If MemoWrite(cArqZip, HttpGet(cUrl))
                    If fUnzip(cArqZip, cDirDow) == 0
                        fErase(cArqZip)
                        ConOut("[D4Sign] PDF assinado salvo em " + cDirDow)
                        // Gravar RH_D4SIGN = '4' na SRH
                    EndIf
                EndIf
            EndIf
        EndIf
    EndIf

Return
```

---

## Padrão de tratamento de erro

Todos os consumidores da classe seguem o mesmo padrão:

```xbase
oD4Sign:<Metodo>(...)
nStatus := oD4Sign:GetStatusCode()

If nStatus >= 200 .And. nStatus < 300
    // Sucesso → processar GetResult(), atualizar SRH, mover arquivo
Else
    // Falha → RH_D4SIGN = '5', gravar GetResult() em RH_IDD4SIG,
    //         mover o PDF para erro\addsigner\ ou erro\sendsigner\
    //         (o job CAD4S005 reprocessa esses diretórios)
EndIf
```

| Etapa que falhou | Destino do PDF | Reprocessado por |
|---|---|---|
| `AddSigner` | `erro\addsigner\` | `CAD4S005 → ReenviaSignatario()` |
| `SendToSign` | `erro\sendsigner\` | `CAD4S005 → ReenviaAssinatura()` |

---
atenção à convenção de nomes (ou migrar para um parâmetro `MV_` de ambiente).
7. **Instância stateful:** `StatusCode` e `Result` refletem apenas a última requisição — em loops, capture os valores antes da próxima chamada.
