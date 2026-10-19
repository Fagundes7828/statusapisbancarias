# Status APIs Bancárias

Painel interno que mostra, em tempo real, a disponibilidade das APIs bancárias usadas pela equipe. Hospedado no **Firebase Hosting**, com acesso restrito por **login Google** às contas `@ixcsoft.com.br`.

**Link:** https://statusapisicoob.web.app

## O que é monitorado

| Banco | Como é verificado | O que aparece |
|---|---|---|
| **Sicoob** – Cobrança Bancária v3 | Chama o endpoint `GET /cobranca-bancaria/v3/health` direto do navegador | Status, latência, código HTTP, módulos internos |
| **Efí Bank** – APIs | Lê a página oficial de status (`status.sejaefi.com.br`, Atlassian Statuspage) | Status oficial, componentes da API, incidentes em andamento |

A página atualiza a cada 30 segundos. O ícone da aba muda de cor (🟢 🟡 🔴), então dá para deixar uma aba fixada e acompanhar.

### Diferença entre os dois tipos

- **`health`** (Sicoob): a página **testa a API de verdade**. Se ela não responder, aparece "Fora do ar".
  - O Sicoob não libera CORS, então o navegador só confirma que a API responde e mede a latência; o corpo (`UP`) e o código HTTP não ficam visíveis.
- **`statuspage`** (Efí): a página mostra **o que o próprio banco informa**. Se o banco cair e ainda não tiver atualizado a página de status, o card continua verde. Se a página de status não puder ser lida, o card mostra "Sem dados", não "Fora do ar".

### Estados possíveis

| Estado | Significado |
|---|---|
| 🟢 Operacional | Respondeu normalmente |
| 🟡 Lento | Respondeu acima de 1,5 s |
| 🟡 Degradado | Algum módulo/componente com problema parcial |
| 🟡 Manutenção | Manutenção programada informada pelo banco |
| 🔴 Fora do ar | Sem resposta, erro HTTP ou falha total informada |
| ⚪ Sem dados | Não foi possível ler a fonte de status |

## Estrutura

```
.
├── public/
│   └── index.html     # página completa (HTML + CSS + JS)
├── firebase.json      # configuração do Hosting
├── .firebaserc        # projeto Firebase (statusapisicoob)
├── .gitignore
└── README.md
```

## Configuração

Tudo fica no bloco `CONFIG`, no começo do `<script>` em `public/index.html`.

### Adicionar um banco

```js
servicos: [
  // Endpoint de health aberto (sem autenticação)
  { nome: "Banco X · Cobrança", url: "https://api.bancox.com.br/health" },

  // Página de status do banco (Atlassian Statuspage)
  { nome: "Banco Y", tipo: "statuspage", url: "https://status.bancoy.com.br", filtro: ["API"] }
]
```

- `filtro`: mostra só componentes cujo nome ou grupo contenha um dos termos. Use `[]` para mostrar todos.
- APIs que exigem token, certificado ou mTLS até para o health (ex.: Bradesco) **não funcionam direto no navegador**; precisariam de um backend guardando as credenciais.

### Liberar acesso

```js
acesso: {
  dominios: ["ixcsoft.com.br"],    // qualquer conta Google desse domínio
  emails: ["fulano@gmail.com"]     // exceções individuais
}
```

### Outros ajustes

| Campo | Padrão | O que faz |
|---|---|---|
| `intervaloSegundos` | 30 | Intervalo entre verificações |
| `timeoutMs` | 10000 | Tempo máximo de espera antes de considerar fora do ar |
| `latenciaLentaMs` | 1500 | Acima disso o serviço aparece como "Lento" |

## Rodar e publicar

Requisitos: Node.js 20+ e Firebase CLI (`npm install -g firebase-tools`).

```bash
firebase login              # uma vez por máquina
firebase serve              # teste local em http://localhost:5000
firebase deploy --only hosting
```

> Abrir o `index.html` com dois cliques não funciona: o SDK do Firebase e a configuração do login são servidos pelo próprio Hosting (`/__/firebase/...`).

### Configuração do projeto Firebase (já feita)

1. Projeto `statusapisicoob` criado no console do Firebase (plano Spark, gratuito).
2. App Web registrado e vinculado ao site do Hosting.
3. **Authentication → Sign-in method → Google** ativado.

Se usar um domínio próprio no futuro, adicione-o em **Authentication → Settings → Domínios autorizados**.

## Problemas comuns

| Sintoma | Solução |
|---|---|
| "O login só funciona com a página servida pelo Firebase Hosting" | Use `firebase serve` ou o link publicado |
| `auth/unauthorized-domain` | Adicione o domínio em Authentication → Domínios autorizados |
| "A conta … não tem acesso" | Confira `CONFIG.acesso` e publique de novo |
| Alteração não aparece após o deploy | `Ctrl + Shift + R` no navegador |
| `Authentication Error: Your credentials are no longer valid` no terminal | `firebase logout`, depois `firebase login --no-localhost` |

## Limitações

- A verificação roda **no navegador de quem está com a página aberta**. Sem ninguém com a página aberta, nada é monitorado e não há alertas automáticos.
- O login protege a tela, mas o HTML é estático e pode ser baixado por quem tiver o link. A página não contém nenhum dado sigiloso.
