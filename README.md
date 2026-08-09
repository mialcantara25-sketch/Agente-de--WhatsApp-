const express = require("express");
const axios = require("axios");
const app = express();
app.use(express.json());
// ─── CONFIGURAÇÕES DO SEU SALÃO ──────────────────────────────────────────────
const Studio = {
  nome: process.env.Studio_NOME || "Studio Michelle Alcantara",
  cidade: process.env.Studio_CIDADE || "Grajau, São Paulo",
  horario: process.env.Studio_HORARIO || "Seg–Sáb, 9h às 19h",
  pagamento: process.env.Studio_PAGAMENTO || "Pix, cartão de crédito/débito, dinheiro",
  tom: process.env.Studio_TOM || "simpática e descontraída",
  boas_vindas: process.env.Studio_BOAS_VINDAS || "Olá! 💚 Seja bem-vinda ao Studio Michelle Alcantara! Sou a Nina, sua atendente virtual. Como posso te ajudar hoje?",
  servicos: [
    { nome: "Alongamento",        preco: "R$ 195"  },
    { nome: "Manutenção de Alongamento",    preco: "R$ 155" },
    { nome: "Banho de gel",    preco: "R$ 140" },
    { nome: "Manutenção de Banho de gel",   preco: "R$ 120"  },
    { nome: "Esmaltação em gel Mãos", preco: "R$ 70"  },
    { nome: "Esmaltação em gel pés",    preco: "R$ 80"  },
    { nome: "Manicure",   preco: "R$ 30"  },
    { nome: "Pedicure", preco: "R$ 45"  },
    { nome: "Manicure + Pedicure",    preco: "R$ 75"  },
  ],
};

// ─── VARIÁVEIS DE AMBIENTE ────────────────────────────────────────────────────
const VERIFY_TOKEN      = process.env.VERIFY_TOKEN;       // token que você inventar
const WHATSAPP_TOKEN    = process.env.WHATSAPP_TOKEN;     // token da Meta
const PHONE_NUMBER_ID   = process.env.PHONE_NUMBER_ID;   // ID do número WhatsApp
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY; // chave da Anthropic

// ─── MEMÓRIA DE CONVERSA (em produção use Redis ou banco de dados) ─────────────
const conversas = {}; // { "numero": [ {role, content}, ... ] }

// ─── SYSTEM PROMPT DA NINA ────────────────────────────────────────────────────
function buildSystemPrompt() {
  const lista = Studio.servicos.map(s => `- ${s.nome}: ${s.preco}`).join("\n");
  return `Você é Nina, atendente virtual ${Studio.tom} do ${Studio.nome}, salão de beleza em ${Studio.cidade}.

HORÁRIO: ${Studio.horario}
PAGAMENTO: ${Studio.pagamento}

SERVIÇOS:
${lista}

SUAS FUNÇÕES:
1. Responder dúvidas sobre serviços, preços e horários
2. Fazer agendamentos (perguntar: serviço, data, horário, nome)
3. Confirmar e cancelar agendamentos
4. Explicar procedimentos (duração, cuidados pré e pós)

REGRAS:
- Seja ${Studio.tom}
- Respostas curtas, máximo 3-4 linhas
- Use emojis com moderação (💅✨)
- Para agendamentos confirme: serviço + data + horário + nome
- Responda SEMPRE em português brasileiro
- Se não souber, diga que vai verificar com a equipe`;
}

// ─── CHAMA A CLAUDE ───────────────────────────────────────────────────────────
async function chamarClaude(numero, mensagem) {
  if (!conversas[numero]) conversas[numero] = [];

  conversas[numero].push({ role: "user", content: mensagem });

  // Mantém só as últimas 20 mensagens para não estourar o contexto
  if (conversas[numero].length > 20) {
    conversas[numero] = conversas[numero].slice(-20);
  }

  const resp = await axios.post(
    "https://api.anthropic.com/v1/messages",
    {
      model: "claude-sonnet-4-6",
      max_tokens: 500,
      system: buildSystemPrompt(),
      messages: conversas[numero],
    },
    {
      headers: {
        "x-api-key": ANTHROPIC_API_KEY,
        "anthropic-version": "2023-06-01",
        "content-type": "application/json",
      },
    }
  );

  const resposta = resp.data.content[0].text;
  conversas[numero].push({ role: "assistant", content: resposta });
  return resposta;
}

// ─── ENVIA MENSAGEM PELO WHATSAPP ─────────────────────────────────────────────
async function enviarWhatsApp(para, texto) {
  await axios.post(
    `https://graph.facebook.com/v19.0/${PHONE_NUMBER_ID}/messages`,
    {
      messaging_product: "whatsapp",
      to: para,
      type: "text",
      text: { body: texto },
    },
    {
      headers: {
        Authorization: `Bearer ${WHATSAPP_TOKEN}`,
        "Content-Type": "application/json",
      },
    }
  );
}

// ─── WEBHOOK: VERIFICAÇÃO (GET) ───────────────────────────────────────────────
app.get("/webhook", (req, res) => {
  const mode      = req.query["hub.mode"];
  const token     = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];

  if (mode === "subscribe" && token === VERIFY_TOKEN) {
    console.log("✅ Webhook verificado pela Meta");
    res.status(200).send(challenge);
  } else {
    res.sendStatus(403);
  }
});

// ─── WEBHOOK: RECEBE MENSAGENS (POST) ─────────────────────────────────────────
app.post("/webhook", async (req, res) => {
  // Confirma recebimento imediatamente (Meta exige resposta em 20s)
  res.sendStatus(200);

  try {
    const entry   = req.body.entry?.[0];
    const changes = entry?.changes?.[0];
    const value   = changes?.value;
    const msg     = value?.messages?.[0];

    if (!msg || msg.type !== "text") return;

    const numero  = msg.from;
    const texto   = msg.text.body.trim();

    console.log(`📩 Mensagem de ${numero}: ${texto}`);

    // Boas-vindas para novos clientes
    const primeiraVez = !conversas[numero];

    const resposta = await chamarClaude(numero, texto);

    const textoFinal = primeiraVez
      ? `${Studio.boas_vindas}\n\n${resposta}`
      : resposta;

    await enviarWhatsApp(numero, textoFinal);
    console.log(`✉️  Resposta para ${numero}: ${textoFinal}`);

  } catch (err) {
    console.error("❌ Erro:", err.response?.data || err.message);
  }
});

// ─── HEALTH CHECK ─────────────────────────────────────────────────────────────
app.get("/", (req, res) => {
  res.json({ status: "online", salao: Studio.nome });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`🚀 Servidor rodando na porta ${PORT}`));
