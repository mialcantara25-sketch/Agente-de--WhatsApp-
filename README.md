Agente de WhatsApp – Studio Michelle Alcantara 💅
Este projeto cria uma atendente virtual chamada Nina, que integra o WhatsApp Business API com o modelo Claude para responder clientes de forma simpática e descontraída.

✨ Funcionalidades
Responde dúvidas sobre serviços, preços e horários

Faz agendamentos (serviço, data, horário, nome)

Confirma e cancela agendamentos

Explica procedimentos e cuidados

🛠️ Tecnologias
Node.js + Express

Axios para chamadas HTTP

Meta WhatsApp Business API

Anthropic Claude API

⚙️ Configuração
Defina as variáveis de ambiente:

VERIFY_TOKEN → token de verificação do webhook

WHATSAPP_TOKEN → token da Meta

PHONE_NUMBER_ID → ID do número WhatsApp

ANTHROPIC_API_KEY → chave da Anthropic

🚀 Executando
bash
npm install
npm start
Servidor rodará em http://localhost:3000.

📌 Endpoints
GET /webhook → verificação da Meta

POST /webhook → recebe mensagens do WhatsApp

GET / → health check
