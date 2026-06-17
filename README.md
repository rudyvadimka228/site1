// netlify/functions/notify.js
//
// Приймає підписки на email та відгуки з сайту і пересилає їх
// у Telegram-бота. Токен бота НІКОЛИ не зберігається в коді —
// він читається зі змінних середовища Netlify (Site settings →
// Environment variables). Дивись README.md для налаштування.

export default async (req) => {
  if (req.method !== "POST") {
    return new Response("Method not allowed", { status: 405 });
  }

  const BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
  const CHAT_ID = process.env.TELEGRAM_CHAT_ID;

  if (!BOT_TOKEN || !CHAT_ID) {
    return new Response(
      JSON.stringify({ error: "Server is not configured" }),
      { status: 500, headers: { "Content-Type": "application/json" } }
    );
  }

  let data;
  try {
    data = await req.json();
  } catch {
    return new Response(JSON.stringify({ error: "Invalid JSON" }), {
      status: 400,
      headers: { "Content-Type": "application/json" },
    });
  }

  let text;
  if (data.type === "subscribe") {
    const email = (data.email || "").toString().trim().slice(0, 200);
    if (!email) {
      return new Response(JSON.stringify({ error: "Email is required" }), {
        status: 400,
        headers: { "Content-Type": "application/json" },
      });
    }
    text = `📩 Нова підписка на розсилку\nEmail: ${email}`;
  } else if (data.type === "review") {
    const name = (data.name || "").toString().trim().slice(0, 100);
    const reviewText = (data.text || "").toString().trim().slice(0, 1000);
    if (!name || !reviewText) {
      return new Response(JSON.stringify({ error: "Name and text are required" }), {
        status: 400,
        headers: { "Content-Type": "application/json" },
      });
    }
    text = `⭐ Новий відгук\nІм'я: ${name}\nТекст: ${reviewText}`;
  } else {
    return new Response(JSON.stringify({ error: "Unknown type" }), {
      status: 400,
      headers: { "Content-Type": "application/json" },
    });
  }

  try {
    const tgRes = await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ chat_id: CHAT_ID, text }),
    });

    if (!tgRes.ok) {
      return new Response(JSON.stringify({ error: "Telegram delivery failed" }), {
        status: 502,
        headers: { "Content-Type": "application/json" },
      });
    }

    return new Response(JSON.stringify({ ok: true }), {
      status: 200,
      headers: { "Content-Type": "application/json" },
    });
  } catch (err) {
    return new Response(JSON.stringify({ error: "Unexpected error" }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
};
