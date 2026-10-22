# 🚀 Automação de Consultoria Financeira com IA e n8n

Este repositório contém o fluxo de automação desenvolvido no **n8n** para processamento, personalização e envio em lote de e-mails de consultoria financeira. A solução integra consumo de dados externos, inteligência artificial generativa (OpenAI), tratamento e higienização de dados em JavaScript e envio final via Gmail com resposta síncrona por Webhook.

---

## 📌 Visão Geral da Arquitetura

O workflow foi projetado para receber uma lista de clientes via API, cruzar com dados atualizados de produtos de investimento, utilizar Inteligência Artificial para redigir uma abordagem humanizada para cada cliente e disparar os e-mails apenas para endereços válidos.

```
[ Webhook ] ──> [ HTTP Request / Code ] ──> [ Merge ] ──> [ Code (Item Normalization) ]
                                                                      │
[ Respond to Webhook ] <── [ Gmail ] <── [ IF (Email Regex) ] <── [ Code (Parser HTML) ] <── [ OpenAI LLM ]
```

---

## 🛠️ Descrição Detalhada dos Nós do Fluxo

### 1. `Webhook` (Gatilho de Entrada)
* **Função:** Ponto de entrada do fluxo. Recebe requisições HTTP `POST /clientes` contendo o payload em JSON com os dados dos clientes (nome, e-mail, saldo e perfil de investidor).

### 2. `HTTP Request` & `Code` (Download e Parsing de Investimentos)
* **Função:** O nó HTTP realiza o download de uma planilha CSV externa contendo a tabela atualizada de produtos de investimento. Em seguida, o nó de Código (JavaScript) converte a estrutura do CSV para um array de objetos JSON manipulável.

### 3. `Merge` (Consolidação dos Dados)
* **Função:** Unifica em uma única estrutura de dados o payload dos clientes (vindo do Webhook) e a lista de produtos de investimento (vinda do HTTP Request/Code).

### 4. `Code` — Normalização de Arrays
* **Função:** Executa a desconstrução (*flattening*) do array de clientes. Transforma uma lista agrupada dentro de `body.clientes` em múltiplos itens individuais nativos do n8n, permitindo o processamento em lote linha por linha nas etapas posteriores.

### 5. `OpenAI` — LLM para Criar E-mails Personalizados (`GPT-4o-mini`)
* **Função:** Atua como um consultor financeiro sênior empático. Para cada cliente, o modelo recebe os atributos (nome, saldo, perfil) e gera um assunto amigável e um texto de e-mail humanizado.
* **Saída:** Retorna um objeto JSON estrito contendo os campos `email`, `assunto` e `mensagem`.

### 6. `Code` — Pré-forma a Mensagem
* **Função:** Sanitização e formatação da resposta da IA.
  * Extrai o texto gerado do payload complexo da OpenAI.
  * Converte a string JSON para um objeto manipulável via `JSON.parse()`.
  * Substitui quebras de linha (`\n`) por tags `<br>` para gerar o corpo do e-mail formatado em HTML (`html_body`).

### 7. `If` — Verifica Autenticação/Sintaxe de E-mail
* **Função:** Nó de validação e higienização (*data cleansing*).
  * Aplica uma **Expressão Regular (Regex)** para validar o formato do e-mail (`user@domain.com`):
    `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
  * **True Branch:** Encaminha apenas os e-mails com sintaxe válida para o envio.
  * **False Branch:** Filtra e descarta e-mails malformatados, evitando erros de execução na API de envio.

### 8. `Gmail` — Envia E-mail Personalizado
* **Função:** Realiza o disparo dos e-mails validados via API do Gmail, preenchendo dinamicamente o destinatário (`to`), assunto (`subject`) e o corpo em HTML (`html_body`). Retorna a confirmação de envio com as *labels* `SENT`.

### 9. `Respond to Webhook` — Retorno para Webhook
* **Função:** Finaliza o ciclo HTTP da requisição inicial, devolvendo o status `200 OK` e o comprovante com os IDs das mensagens enviadas no Gmail ao sistema requisitante.

---

## ⚙️ Tecnologias e Recursos Utilizados

* **Orquestrador:** n8n (Self-hosted / Cloud)
* **Linguagem dos Scripts:** JavaScript (Node.js no contexto do n8n)
* **Modelos de IA:** OpenAI GPT-4o-mini
* **Validação:** Expressões Regulares (Regex)
* **Integrações:** Webhook HTTP, Gmail API

---

## 💡 Como Importar e Rodar este Fluxo

1. Faça o download ou copie o arquivo JSON do fluxo localizado neste repositório.
2. No seu painel do n8n, clique em **Workflows** > **Import from File** (ou colar do clipboard).
3. **Configuração de Credenciais:**
   * Configure suas credenciais da **OpenAI API Key**.
   * Configure a autenticação OAuth2 da sua conta do **Gmail**.
4. Ative o nó **Webhook** e envie uma requisição `POST` para testar o disparo.