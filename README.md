# Serverless n8n Blog Pipeline
**How to Deploy a 100% Free, Always-On n8n Automation Server (Hugging Face + Supabase + CronJob)**

Running a self-hosted n8n instance usually means paying for a VPS. Relying on free ephemeral cloud containers usually means losing all your workflows the moment the server spins down.

In this guide, I will show you how to architect a completely free, stateless, always-on n8n environment. By decoupling the storage from the compute and utilizing edge proxies, you can build an enterprise-grade automation engine without spending a dime.

## The Architecture Overview
To achieve a zero-cost, persistent setup, we will use four services:

1. **Supabase (PostgreSQL):** To permanently store our n8n workflows, credentials, and execution logs.
2. **Hugging Face Spaces:** To host the n8n Docker container for free.
3. **Cron-job.org:** To prevent the container from sleeping.
4. **Cloudflare Workers:** To proxy external requests and bypass strict outbound firewalls.

---

## Step 1: Set Up the Persistent Database (Supabase)
Hugging Face wipes its local hard drive every time the container restarts. We must point n8n to an external database so our data survives reboots.

1. Create a free project on [Supabase](https://supabase.com/). Name it something like `n8n-database` and save the database password securely.
2. Go to **Settings > Database**.

> [!IMPORTANT]
> Hugging Face struggles with IPv6 outbound traffic. You must use Supabase's Session Pooler (IPv4) connection string. Note that the pooler port is typically `6543`.

3. Copy the connection details (Host, Port `6543`, User, and Password).

---

## Step 2: Deploy the Engine (Hugging Face)
Hugging Face offers free Docker hosting via their "Spaces" feature.

1. Create a free Hugging Face account and create a new **Space**.
2. Select **Docker** as the SDK and **Blank** as the template.

> [!CAUTION]
> Set the Space visibility to **Private**. If it is public, your webhook URLs and login screen will be exposed to the internet.

3. Create a `Dockerfile` in the repository and paste this exact configuration:

```dockerfile
FROM n8nio/n8n:latest

ENV N8N_PORT=7860
ENV N8N_LISTEN_ADDRESS=0.0.0.0
EXPOSE 7860

CMD ["start"]
```

---

## Step 3: Configure the Environment Variables
Before building the Space, you need to map your Supabase database and secure the instance. Go to your Space's **Settings > Variables and secrets** and add the following as **Secrets**:

**Database Connection:**
- `DB_TYPE` = `postgresdb`
- `DB_POSTGRESDB_HOST` = *(Your Supabase IPv4 Pooler URL)*
- `DB_POSTGRESDB_PORT` = `6543` *(or `5432` if you have the IPv4 add-on)*
- `DB_POSTGRESDB_DATABASE` = `postgres`
- `DB_POSTGRESDB_USER` = *(Your Supabase User)*
- `DB_POSTGRESDB_PASSWORD` = *(Your Supabase Password)*

**Security & Routing:**
- `N8N_ENCRYPTION_KEY` = *(A long, random string. Tip: Generate a UUID. If you do not set this, n8n generates a new key on every reboot, permanently locking you out of your saved API credentials).*
- `WEBHOOK_URL` = `https://your-username-your-spacename.hf.space`
- `N8N_PROXY_HOPS` = `1`

Click **Restart this Space**. When it boots, it will automatically connect to Supabase, build its tables, and provide you with the setup screen at your Space's URL.

---

## Step 4: The Keep-Alive Protocol (Cron-job.org)
Hugging Face will automatically put free Spaces to sleep after 48 hours of no web traffic. Scheduled internal n8n triggers do not count as web traffic. We have to ping the server externally to keep it awake.

Because your Space is Private, a standard ping will be blocked by Hugging Face's security proxy. We need to give the ping a VIP pass.

1. In Hugging Face, go to your **Profile Settings > Access Tokens** and create a new **Read** token. Copy it.
2. Create a free account at [cron-job.org](https://cron-job.org) and create a new cronjob.
3. **URL:** Point it specifically to n8n's health endpoint: `https://your-username-your-spacename.hf.space/healthz`
4. **Schedule:** Every 15 minutes.
5. **Advanced Settings:** Add a custom HTTP Header:
   - **Name:** `Authorization`
   - **Value:** `Bearer [YOUR_HUGGING_FACE_TOKEN]`

Save and test. You should receive a `200 OK` response with `{"status":"ok"}`. Your server will now stay awake 24/7.

---

## Step 5: Bypassing Network Firewalls (Cloudflare Workers)
Hugging Face intentionally blocks outbound traffic to social APIs (like Telegram, WhatsApp, and Facebook) to prevent spam bots. If your n8n workflows rely on these nodes, the connections will fail.

You can bypass this by routing the request through a free Cloudflare Worker.

1. In Cloudflare, create a new **Worker** (e.g., `telegram-proxy`).
2. Replace the default `worker.js` code with this proxy script:

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const targetUrl = `https://api.telegram.org${url.pathname}${url.search}`;

    if (request.method !== "POST") return new Response("Method Not Allowed", { status: 405 });

    const modifiedRequest = new Request(targetUrl, {
      method: request.method,
      headers: request.headers,
      body: request.body,
      redirect: "follow"
    });

    try {
      const response = await fetch(modifiedRequest);
      return new Response(response.body, response);
    } catch (error) {
      return new Response(error.message, { status: 500 });
    }
  },
};
```
*(Note: This script strictly allows POST requests, which is perfect for sending messages. If your workflow needs to fetch data using GET, remove the POST restriction).*

3. Deploy the Worker and copy its URL.
4. In n8n, delete your native Telegram node and replace it with a standard **HTTP Request** node.
5. Set the URL to: `https://your-worker.workers.dev/bot[YOUR_BOT_TOKEN]/sendMessage`
6. Send the payload as a JSON body containing your `chat_id` and `text`.

Hugging Face allows the outbound request to Cloudflare, and Cloudflare seamlessly hands the payload to Telegram.

---

## Step 6: Free AI Compute (Optional)
If your automations rely heavily on LLMs, standard OpenAI API keys will get expensive. Instead, utilize **GitHub Models**.

GitHub provides a free inference API that is 100% compatible with OpenAI's format.

1. Generate a GitHub **Personal Access Token**.
2. In n8n, create an OpenAI Credential but override the Base URL to `https://models.inference.ai.azure.com`.
3. Use your GitHub token as the API key.

> [!NOTE]
> Be aware of strict rate limits (e.g., 1 request per minute for heavy models like DeepSeek-R1). Use Wait nodes in n8n to stagger your AI calls if necessary.

---

## Conclusion
By combining these free tiers, you now have a highly resilient, stateless automation engine. You can safely deploy complex, multi-agent architectures without the monthly overhead of a dedicated cloud server.
