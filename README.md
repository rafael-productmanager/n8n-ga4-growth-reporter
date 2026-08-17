# 📊 Multi-Site GA4 Analytics & Growth Pipeline (n8n + Slack)

Pipeline automatizado de inteligência de tráfego construído no **n8n**. O fluxo extrai semanalmente métricas de aquisição e retenção de múltiplos produtos SaaS e sites institucionais via **Google Analytics Data API (GA4)**, normaliza os dados via JavaScript e entrega um relatório acionável diretamente em canal privado do **Slack**.

---

## 🎯 Contexto e Objetivo de Produto

Em operações com múltiplos produtos e domínios, o acompanhamento de métricas costuma sofrer com dois extremos:
1. **Fricção operacional:** Necessidade de alternar entre diferentes propriedades no dashboard do GA4.
2. **Métricas de vaidade:** Alertas que mostram apenas volume bruto de acessos, sem discriminar canal de origem nem taxa real de engajamento.

Este fluxo automatiza a coleta centralizada e entrega uma visão enxuta de **qualidade de tráfego por canal** para direcionar esforços semanais de SEO, tráfego e otimização de conversão (CRO).

---

## 🏗️ Arquitetura do Fluxo

```text
[Schedule Trigger] 
       │ (Semanal)
       ▼
[GA4 - Produto 1] ──► [GA4 - Produto 2] ──► [GA4 - Produto 3]
                                                   │
                                                   ▼
                                         [Code Node (JS)]
                                      (Normalização & Layout)
                                                   │
                                                   ▼
                                         [HTTP Request - Slack]
                                           (Incoming Webhook)

```

---

## 💡 Decisões de Arquitetura & Trade-offs

Durante a estruturação do pipeline no n8n, uma decisão técnica relevante foi tomada quanto à consolidação dos dados:

### 1. Merge Node em Paralelo vs. Execução em Série (Linear)

* **Abordagem Inicial (Merge em Paralelo):** Disparar as 3 requisições HTTP do GA4 em paralelo a partir do gatilho e uni-las com um nó `Merge`.
* **Problema Encontrado:** O nó `Merge` nativo limita a combinação direta a 2 entradas por nó (`Input 1` e `Input 2`), exigindo encadeamento de múltiplos nós de mesclagem ou gerando perda de contexto entre os itens combinados.
* **Solução Adotada (Execução em Série + Code Node):** Encadear os nós HTTP em série (`Gatilho` $\rightarrow$ `GA4 1` $\rightarrow$ `GA4 2` $\rightarrow$ `GA4 3` $\rightarrow$ `Code`).
* **Vantagem:** O nó `Code` consegue acessar os dados de qualquer nó anterior pelo seletor `$('Nome do Nó').first().json`, eliminando nós intermediários de merge e tornando o pipeline mais limpo, previsível e fácil de escalar para novos produtos.

---

## ⚙️ Detalhamento dos Nós

### 1. Gatilho (`Schedule Trigger`)

* **Frequência:** Semanal (Segundas-feiras, 08:00).
* Inicia o ciclo de extração automatizada antes das reuniões de alinhamento da equipe.

### 2. Extração (`Google Analytics Data API v1beta`)

* **Método:** `POST`
* **Endpoint:** `https://analyticsdata.googleapis.com/v1beta/properties/{PROPERTY_ID}:runReport`
* **Autenticação:** Google Service Account (OAuth 2.0 com escopo `analytics.readonly`).
* **Dimensões & Métricas Consultadas:**
* Dimensão: `sessionDefaultChannelGroup` (Origem do tráfego).
* Métricas: `sessions`, `activeUsers`, `engagementRate`, `eventCount`.
* Janela temporal: `7daysAgo` até `yesterday`.

### 3. Processamento & Formatação (`Code in JavaScript`)

* Itera sobre os canais retornados de cada propriedade.
* Calcula total de usuários únicos, sessões e percentual de engajamento por canal (`Organic Search`, `Direct`, `Referral`, `Paid Search`, `AI Assistant`).
* Trata propriedades sem acessos recentes sem quebrar a execução do fluxo.
* Monta a estrutura da mensagem com markdown compatível com o Slack.

### 4. Disparo (`HTTP Request - Slack`)

* **Método:** `POST`
* **Destino:** Slack Incoming Webhook (`/services/...`)
* **Payload:**
```json
{
  "text": {{ JSON.stringify($json.textoMensagem) }}
}

```



---

## 💻 Script de Processamento (JavaScript)

```javascript
const formatReport = (nodeName, siteTitle) => {
  const rows = $(nodeName).first()?.json?.rows || [];
  if (!rows.length) return `*${siteTitle}*\n• Sem acessos registrados no período.\n`;

  let totalSessions = 0;
  let totalUsers = 0;
  let channels = [];

  rows.forEach(r => {
    const channel = r.dimensionValues[0].value;
    const sessions = parseInt(r.metricValues[0].value, 10);
    const users = parseInt(r.metricValues[1].value, 10);
    const engagement = (parseFloat(r.metricValues[2].value) * 100).toFixed(0);

    totalSessions += sessions;
    totalUsers += users;
    channels.push(`  ▫ ${channel}: ${sessions} sessões (Engajamento: ${engagement}%)`);
  });

  return `*${siteTitle}*
• Total: ${totalUsers} usuários | ${totalSessions} sessões
• Canais de Aquisição:
${channels.join('\n')}\n`;
};

const relatorioProduto1 = formatReport('GA4 - Product 1', '🌳 Produto 1');
const relatorioProduto2 = formatReport('GA4 - Product 2', '💻 Produto 2');
const relatorioProduto3 = formatReport('GA4 - Product 3', '📍 Produto 3');

const textoMensagem = `📊 *REPORTE SEMANAL DE AQUISIÇÃO & TRÁFEGO*
Período: Últimos 7 dias

${relatorioProduto1}
${relatorioProduto2}
${relatorioProduto3}
---
_Foco semanal: Acompanhar evolução de SEO Orgânico e Engajamento da LP._`;

return [{ json: { textoMensagem } }];

```

---

## 📱 Exemplo de Notificação Entregue no Slack

```text
📊 REPORTE SEMANAL DE AQUISIÇÃO & TRÁFEGO
Período: Últimos 7 dias

🌳 Produto 1
• Total: 23 usuários | 24 sessões
• Canais de Aquisição:
  ▫ Direct: 10 sessões (Engajamento: 0%)
  ▫ Organic Search: 7 sessões (Engajamento: 86%)
  ▫ Referral: 4 sessões (Engajamento: 50%)
  ▫ Paid Search: 2 sessões (Engajamento: 0%)
  ▫ AI Assistant: 1 sessões (Engajamento: 100%)

💻 Produto 2
• Sem acessos registrados no período.

📍 Produto 3
• Sem acessos registrados no período.

---
Foco semanal: Acompanhar evolução de SEO Orgânico e Engajamento da LP.

```

---

## 🛠️ Tecnologias Utilizadas

* **Orquestrador:** n8n
* **Fonte de Dados:** Google Analytics 4 (Data API v1beta)
* **Transformação de Dados:** JavaScript (Node.js no n8n)
* **Canal de Notificação:** Slack API (Incoming Webhooks)
* **Segurança:** GCP Service Account (OAuth 2.0 / escopos mínimos de leitura)

```

4. Clique no botão verde **Commit changes...** no canto superior direito e confirme clicando em **Commit changes**.

---

**Passo 3: Criar o arquivo `workflow.json`**

1. De volta à página principal do repositório, clique no botão **Add file** $\rightarrow$ selecione **Create new file**.
2. No campo **Name your file...**, digite exatamente: `workflow.json`.
3. No campo de texto grande abaixo, copie e cole todo o conteúdo deste bloco:

```json
{
  "name": "Multi-Site GA4 Analytics & Growth Pipeline",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "weeks",
              "triggerAtDay": [
                1
              ],
              "triggerAtHour": 8
            }
          ]
        }
      },
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.3,
      "position": [
        -1152,
        96
      ],
      "id": "cba89bb4-79b0-4b5b-a996-0d0736457486",
      "name": "Schedule Trigger"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://analyticsdata.googleapis.com/v1beta/properties/YOUR_GA4_PROPERTY_ID_1:runReport",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "googleApi",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "{\n  \"dateRanges\": [\n    {\n      \"startDate\": \"7daysAgo\",\n      \"endDate\": \"yesterday\"\n    }\n  ],\n  \"dimensions\": [\n    { \"name\": \"sessionDefaultChannelGroup\" }\n  ],\n  \"metrics\": [\n    { \"name\": \"sessions\" },\n    { \"name\": \"activeUsers\" },\n    { \"name\": \"engagementRate\" },\n    { \"name\": \"eventCount\" }\n  ],\n  \"limit\": 5\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.5,
      "position": [
        -880,
        96
      ],
      "id": "a3682c0d-97bc-4727-805d-a44a7008f82b",
      "name": "GA4 - Product 1",
      "credentials": {
        "googleServiceAccountApi": {
          "id": "YOUR_GOOGLE_SERVICE_ACCOUNT_CREDENTIAL_ID",
          "name": "Google Service Account"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://analyticsdata.googleapis.com/v1beta/properties/YOUR_GA4_PROPERTY_ID_2:runReport",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "googleApi",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "{\n  \"dateRanges\": [\n    {\n      \"startDate\": \"7daysAgo\",\n      \"endDate\": \"yesterday\"\n    }\n  ],\n  \"dimensions\": [\n    { \"name\": \"sessionDefaultChannelGroup\" }\n  ],\n  \"metrics\": [\n    { \"name\": \"sessions\" },\n    { \"name\": \"activeUsers\" },\n    { \"name\": \"engagementRate\" },\n    { \"name\": \"eventCount\" }\n  ],\n  \"limit\": 5\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.5,
      "position": [
        -640,
        96
      ],
      "id": "37145a10-9178-4b4b-8db1-3157d19da33c",
      "name": "GA4 - Product 2",
      "credentials": {
        "googleServiceAccountApi": {
          "id": "YOUR_GOOGLE_SERVICE_ACCOUNT_CREDENTIAL_ID",
          "name": "Google Service Account"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://analyticsdata.googleapis.com/v1beta/properties/YOUR_GA4_PROPERTY_ID_3:runReport",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "googleApi",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "{\n  \"dateRanges\": [\n    {\n      \"startDate\": \"7daysAgo\",\n      \"endDate\": \"yesterday\"\n    }\n  ],\n  \"dimensions\": [\n    { \"name\": \"sessionDefaultChannelGroup\" }\n  ],\n  \"metrics\": [\n    { \"name\": \"sessions\" },\n    { \"name\": \"activeUsers\" },\n    { \"name\": \"engagementRate\" },\n    { \"name\": \"eventCount\" }\n  ],\n  \"limit\": 5\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.5,
      "position": [
        -416,
        96
      ],
      "id": "933123bb-2e84-4621-a272-32e561399e0d",
      "name": "GA4 - Product 3",
      "credentials": {
        "googleServiceAccountApi": {
          "id": "YOUR_GOOGLE_SERVICE_ACCOUNT_CREDENTIAL_ID",
          "name": "Google Service Account"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "const formatReport = (nodeName, siteTitle) => {\n  const rows = $(nodeName).first()?.json?.rows || [];\n  if (!rows.length) return `*${siteTitle}*\\n• Sem acessos registrados no período.\\n`;\n\n  let totalSessions = 0;\n  let totalUsers = 0;\n  let channels = [];\n\n  rows.forEach(r => {\n    const channel = r.dimensionValues[0].value;\n    const sessions = parseInt(r.metricValues[0].value, 10);\n    const users = parseInt(r.metricValues[1].value, 10);\n    const engagement = (parseFloat(r.metricValues[2].value) * 100).toFixed(0);\n\n    totalSessions += sessions;\n    totalUsers += users;\n    channels.push(`  ▫ ${channel}: ${sessions} sessões (Engajamento: ${engagement}%)`);\n  });\n\n  return `*${siteTitle}*\\n• Total: ${totalUsers} usuários | ${totalSessions} sessões\\n• Canais de Aquisição:\\n${channels.join('\\n')}\\n`;\n};\n\nconst relatorioProduto1 = formatReport('GA4 - Product 1', '🌳 Product 1');\nconst relatorioProduto2 = formatReport('GA4 - Product 2', '💻 Product 2');\nconst relatorioProduto3 = formatReport('GA4 - Product 3', '📍 Product 3');\n\nconst textoMensagem = `📊 *REPORTE SEMANAL DE AQUISIÇÃO & TRÁFEGO*\\nPeríodo: Últimos 7 dias\\n\\n${relatorioProduto1}\\n${relatorioProduto2}\\n${relatorioProduto3}\\n---\\n_Foco semanal: Acompanhar evolução de SEO Orgânico e Engajamento da LP._`;\n\nreturn [{ json: { textoMensagem } }];"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -176,
        96
      ],
      "id": "cfe5ac9c-a9d1-470e-90e5-30079be0af82",
      "name": "Code in JavaScript"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://hooks.slack.com/services/YOUR_SLACK_WEBHOOK_URL_HERE",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"text\": {{ JSON.stringify($json.textoMensagem) }}\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.5,
      "position": [
        48,
        96
      ],
      "id": "ffbd9491-565b-4a13-94d8-854e382ac953",
      "name": "HTTP Request"
    }
  ],
  "pinData": {},
  "connections": {
    "Schedule Trigger": {
      "main": [
        [
          {
            "node": "GA4 - Product 1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "GA4 - Product 1": {
      "main": [
        [
          {
            "node": "GA4 - Product 2",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "GA4 - Product 2": {
      "main": [
        [
          {
            "node": "GA4 - Product 3",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "GA4 - Product 3": {
      "main": [
        [
          {
            "node": "Code in JavaScript",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code in JavaScript": {
      "main": [
        [
          {
            "node": "HTTP Request",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request": {
      "main": [
        []
      ]
    }
  },
  "active": true,
  "settings": {
    "executionOrder": "v1",
    "binaryMode": "separate"
  },
  "tags": []
}

```

4. Clique no botão verde **Commit changes...** e confirme clicando em **Commit changes**.
                                           
