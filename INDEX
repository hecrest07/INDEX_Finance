// MisFinanzas Bot Worker
// Conecta Telegram → Gemini AI (gratis) → Google Sheets

const express = require('express');
const axios = require('axios');
const { google } = require('googleapis');
const { GoogleGenerativeAI } = require('@google/generative-ai');
const app = express();
app.use(express.json({ limit: '10mb' }));

// ── Clientes ──────────────────────────────────────────────
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);
const model = genAI.getGenerativeModel({ model: 'gemini-2.0-flash' });
const TELEGRAM_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
const SHEET_ID = process.env.GOOGLE_SHEET_ID;
const TELEGRAM_API = `https://api.telegram.org/bot${TELEGRAM_TOKEN}`;

// ── Google Sheets Auth ────────────────────────────────────
const auth = new google.auth.GoogleAuth({
  credentials: JSON.parse(process.env.GOOGLE_SERVICE_ACCOUNT),
  scopes: ['https://www.googleapis.com/auth/spreadsheets'],
});

async function getSheets() {
  const client = await auth.getClient();
  return google.sheets({ version: 'v4', auth: client });
}

// ── Categorías predefinidas ───────────────────────────────
const CATEGORIES = [
  'comida', 'transporte', 'entretenimiento',
  'salud', 'ropa', 'servicios',
  'supermercado', 'viajes', 'educacion', 'otros'
];

// ── Leer transacciones existentes (anti-duplicados) ───────
async function getExistingTransactions() {
  try {
    const sheets = await getSheets();
    const res = await sheets.spreadsheets.values.get({
      spreadsheetId: SHEET_ID,
      range: 'Transacciones!A2:F',
    });
    return res.data.values || [];
  } catch(e) { return []; }
}

// ── Detectar duplicados ───────────────────────────────────
function isDuplicate(existing, newTx) {
  return existing.some(row => {
    const sameDesc = row[2]?.toLowerCase() === newTx.desc.toLowerCase();
    const sameAmt  = parseFloat(row[3]) === parseFloat(newTx.amount);
    const dayDiff  = Math.abs(new Date(row[0]) - new Date(newTx.date)) / 86400000;
    return sameDesc && sameAmt && dayDiff < 2;
  });
}

// ── Insertar en Google Sheets ─────────────────────────────
async function appendTransaction(tx) {
  const sheets = await getSheets();
  await sheets.spreadsheets.values.append({
    spreadsheetId: SHEET_ID,
    range: 'Transacciones!A:G',
    valueInputOption: 'USER_ENTERED',
    resource: {
      values: [[
        tx.date, tx.type, tx.desc,
        tx.amount, tx.category,
        tx.account || 'Bot Telegram', tx.source
      ]]
    },
  });
}

// ── Parsear JSON de respuesta de Gemini ───────────────────
function parseGeminiJSON(text) {
  const clean = text.replace(/```json|```/g, '').trim();
  return JSON.parse(clean);
}

// ── Procesar imagen de ticket con Gemini Vision ───────────
async function processReceiptImage(fileUrl) {
  const imgRes = await axios.get(fileUrl, { responseType: 'arraybuffer' });
  const base64 = Buffer.from(imgRes.data).toString('base64');
  const mimeType = imgRes.headers['content-type'] || 'image/jpeg';

  const result = await model.generateContent([
    {
      inlineData: { data: base64, mimeType }
    },
    `Analiza este ticket/recibo y extrae la información.
Responde SOLO con JSON, sin texto extra, sin markdown:
{
  "desc": "nombre del comercio o establecimiento",
  "amount": 0.00,
  "date": "YYYY-MM-DD",
  "category": "una de: ${CATEGORIES.join(', ')}",
  "type": "gasto"
}
Si no ves una fecha clara, usa hoy: ${new Date().toISOString().split('T')[0]}`
  ]);

  return parseGeminiJSON(result.response.text());
}

// ── Procesar texto libre con Gemini ──────────────────────
async function processTextMessage(text) {
  const result = await model.generateContent(
    `El usuario registra un movimiento financiero: "${text}"
Extrae la información. Responde SOLO con JSON sin markdown:
{
  "desc": "descripción clara del gasto o ingreso",
  "amount": 0.00,
  "date": "${new Date().toISOString().split('T')[0]}",
  "category": "una de: ${CATEGORIES.join(', ')}",
  "type": "gasto o ingreso"
}
Reglas: si menciona sueldo/pago/depósito/cobro → type ingreso. Si no hay monto claro → amount null.`
  );

  return parseGeminiJSON(result.response.text());
}

// ── Enviar mensaje a Telegram ─────────────────────────────
async function sendTelegram(chatId, text, parse = 'Markdown') {
  await axios.post(`${TELEGRAM_API}/sendMessage`, {
    chat_id: chatId, text, parse_mode: parse
  });
}

// ── Webhook principal ─────────────────────────────────────
app.post('/webhook', async (req, res) => {
  res.sendStatus(200);
  const msg = req.body.message;
  if (!msg) return;
  const chatId = msg.chat.id;

  try {
    let tx = null;

    // 📸 Imagen → ticket
    if (msg.photo) {
      await sendTelegram(chatId, '📸 Analizando tu ticket con Gemini...');
      const photo = msg.photo[msg.photo.length - 1];
      const fileRes = await axios.get(`${TELEGRAM_API}/getFile?file_id=${photo.file_id}`);
      const fileUrl = `https://api.telegram.org/file/bot${TELEGRAM_TOKEN}/${fileRes.data.result.file_path}`;
      tx = await processReceiptImage(fileUrl);
      tx.source = 'ticket_foto';
    }

    // 📄 PDF → estado de cuenta (Fase 3)
    else if (msg.document?.mime_type === 'application/pdf') {
      await sendTelegram(chatId, '📄 Procesamiento de estados de cuenta PDF disponible en Fase 3.');
      return;
    }

    // 💬 Texto libre
    else if (msg.text) {
      const text = msg.text.trim();
      if (text === '/start') {
        await sendTelegram(chatId,
          `🤑 *MisFinanzas Bot activo*\n\n` +
          `Powered by *Gemini AI* (gratis ✓)\n\n` +
          `Puedes enviarme:\n` +
          `• 📸 Foto de tu ticket\n` +
          `• 💬 Texto: \`150 uber\` o \`ingreso 18000 sueldo\`\n` +
          `• 📄 PDF del estado de cuenta BBVA (próximamente)`
        );
        return;
      }
      await sendTelegram(chatId, '🧠 Clasificando con Gemini...');
      tx = await processTextMessage(text);
      tx.source = 'texto_manual';
    }

    if (!tx || !tx.amount) {
      await sendTelegram(chatId, '❓ No pude identificar el monto. Intenta: `150 uber` o sube una foto del ticket.');
      return;
    }

    // 🔍 Anti-duplicados
    const existing = await getExistingTransactions();
    if (isDuplicate(existing, tx)) {
      await sendTelegram(chatId,
        `⚠️ *Posible duplicado detectado*\n\n` +
        `Ya existe: ${tx.desc} — $${tx.amount}\n\n` +
        `No se guardó para evitar duplicados.`
      );
      return;
    }

    // 💾 Guardar en Sheets
    await appendTransaction(tx);
    const emoji = tx.type === 'ingreso' ? '💚' : '🔴';
    await sendTelegram(chatId,
      `${emoji} *${tx.type.toUpperCase()} registrado*\n\n` +
      `📝 ${tx.desc}\n` +
      `💰 $${parseFloat(tx.amount).toLocaleString('es-MX')}\n` +
      `🏷️ ${tx.category}\n` +
      `📅 ${tx.date}\n\n` +
      `_✓ Guardado en Google Sheets_`
    );

  } catch(err) {
    console.error(err);
    await sendTelegram(chatId, '❌ Error procesando tu mensaje. Intenta de nuevo.');
  }
});

app.get('/', (req, res) => res.send('MisFinanzas Bot con Gemini ✓'));
app.listen(3000, () => console.log('Bot escuchando en puerto 3000'));
