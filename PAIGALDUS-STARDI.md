# stardi.me — spämmi- ja maksekaitse paigaldus

**Probleem, mis sai lahendatud:** varem saadeti tellimuskiri kohe vormi täitmisel,
enne makset. Nüüd väljub kiri **alles pärast kinnitatud Stripe makset**. Maksmata
jätja ei käivita ühtegi e-kirja.

Sammud on ühekordsed (~15 min). Pärast seda toimib kõik automaatselt.

---

## 1. Deploy'i Cloudflare Worker

Kaustas `worker/` on `worker.js` ja `wrangler.toml`.

```bash
cd worker
npm install -g wrangler          # kui pole
wrangler login
wrangler secret put STRIPE_SECRET_KEY      # kleebi sk_live_... (Stripe → Developers → API keys)
wrangler secret put WEB3FORMS_KEY          # sinu olemasolev access_key: 908f25a7-022f-4318-9c81-ad416e0d176c
wrangler deploy
```

Deploy annab sulle aadressi, nt:
`https://stardi-checkout.sinu-alamdomeen.workers.dev`

> `STRIPE_WEBHOOK_SECRET`-i paned 3. sammus.

---

## 2. Ühenda vorm Workeriga

Ava `index.html`, leia rida:

```js
const CHECKOUT_ENDPOINT='https://stardi-checkout.SINU-ALAMDOMEEN.workers.dev/create-checkout';
```

Asenda `SINU-ALAMDOMEEN` oma tegeliku Workeri aadressiga (1. sammust). Lõpus peab
jääma `/create-checkout`.

---

## 3. Loo Stripe webhook

Stripe Dashboard → **Developers → Webhooks → Add endpoint**

- **Endpoint URL:** `https://stardi-checkout.sinu-alamdomeen.workers.dev/webhook`
- **Events to send:** vali ainult `checkout.session.completed`
- Salvesta. Stripe näitab **Signing secret** kujul `whsec_...`

Pane see Workerisse:

```bash
cd worker
wrangler secret put STRIPE_WEBHOOK_SECRET   # kleebi whsec_...
wrangler deploy
```

---

## 4. Laadi sait GitHubi

Pane muudetud `index.html` (koos honeypot-väljaga) GitHubi tavalisel viisil.
Muid faile muutma ei pea — `aitah.html` jääb samaks (Stripe suunab sinna
`?ok=1`, kiri on selleks hetkeks juba webhookist saadetud).

---

## Kuidas nüüd toimib

1. Klient täidab vormi → Worker loob Stripe makse → suunatakse maksele
2. **Makse õnnestub** → Stripe kutsub `/webhook` → Worker saadab sulle kirja
   pealkirjaga **„TASUTUD tellimus: …"**
3. Klient jõuab `aitah.html` lehele
4. Kui klient ei maksa → sulle ei tule **mitte midagi**

## Kaitsekihid

- E-kiri ainult pärast kinnitatud makset (põhikaitse)
- Stripe webhooki allkirja kontroll (keegi ei saa võltskirja tekitada)
- Honeypot-väli + „liiga kiire täitmine" kontroll (robotid)
- CORS lukustatud ainult `https://stardi.me` peale

## Test enne live'i

Kasuta esmalt Stripe **test-võtmeid** (`sk_test_...`, test-webhook). Testkaart:
`4242 4242 4242 4242`, suvaline tulevikukuupäev ja CVC. Kui testkirja saad kätte,
vaheta live-võtmete vastu.

## NB
- Ära vajuta vanadele „kiisu"-tüüpi kirjadele **Report Spam** — see võib panna
  Gmaili ka päris tellimusi rämpsuks pidama. Lihtsalt kustuta.
- Sama Workerit saad hiljem laiendada (nt Telegrami teavitus), aga praeguse
  probleemi see lahendab täielikult.
