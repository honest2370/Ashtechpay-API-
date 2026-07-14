# Ashtech Pay — Hosted Payment Page API Documentation

**Version:** v1 (Mai 2026)

Créez des liens de paiement hébergés et acceptez des paiements Mobile Money dans 22+ pays africains, sans gérer vous-même la page de paiement.

| | |
|---|---|
| **Endpoint principal** | `POST /api/v1/hosted-payment/create` |
| **Authentification** | `Bearer hp_live_xxxxxxxx` |
| **Base URL** | `https://ashtechpay.top` |
| **Version** | v1 — Mai 2026 |
| **Contact** | support@ashtechpay.top |

## Sommaire

1. [Les 3 clés API](#1-les-3-clés-api)
2. [Créer un lien de paiement — POST /api/v1/hosted-payment/create](#2-créer-un-lien-de-paiement)
3. [Prix fixe — montant défini à l'avance](#3-prix-fixe--tu-définis-le-montant)
4. [Prix libre — le client choisit le montant](#4-prix-libre--le-client-choisit-le-montant)
5. [Filtrer les pays affichés (22 pays disponibles)](#5-filtrer-les-pays-affichés)
6. [Vérifier le statut — GET /api/v1/hosted-payment/:payment_id](#6-vérifier-le-statut-dun-paiement)
7. [Créditement automatique du wallet](#7-créditement-automatique-du-wallet)
8. [Webhook — notification automatique](#8-webhook--notification-automatique)
9. [Exemples de code (Node.js · PHP · cURL)](#9-exemples-de-code)

---

## 1. Les 3 clés API

En générant tes clés dans l'onglet **Hosted Page**, tu obtiens 3 clés distinctes :

| Clé | Préfixe | Rôle | Où l'utiliser |
|---|---|---|---|
| Public Key | `pk_live_` | Identification publique | Frontend JS — identifie ton compte côté client |
| Secret Key | `sk_live_` | Opérations sensibles | Backend uniquement — webhooks, remboursements |
| Hosted Page Key | `hp_live_` | Liens de paiement hébergés | Backend — créer des liens via API |

> ⚠️ **Sécurité** — `sk_live_` et `hp_live_` doivent rester dans des variables d'environnement côté serveur. Ne les publie jamais dans du code frontend ni dans un dépôt Git public.

---

## 2. Créer un lien de paiement

`POST /api/v1/hosted-payment/create`

Crée un lien de paiement hébergé unique. Le client est redirigé vers une page Ashtech Pay sécurisée pour finaliser le paiement Mobile Money.

### En-têtes requis

```http
Authorization: Bearer hp_live_xxxxxxxxxxxxxxxxxxxxxxxx
Content-Type: application/json
```

### Corps de la requête (JSON)

| Paramètre | Type | Statut | Description |
|---|---|---|---|
| `currency` | string | Requis | Devise : XOF, XAF, GNF, CDF… |
| `amount` | number | Optionnel | Montant fixe. Obligatoire si `is_fixed_amount` est `true` |
| `description` | string | Optionnel | Titre affiché sur la page de paiement |
| `is_fixed_amount` | boolean | Optionnel | `true` (défaut) = prix fixe. `false` = client saisit le montant |
| `allowed_countries` | string[] | Optionnel | Codes ISO des pays à afficher. Ex: `["CM","SN"]`. Vide = tous |
| `notify_url` | string | Optionnel | Surcharge la Webhook URL configurée pour ce lien spécifique |

### Réponse 200 (succès)

```json
{
  "status": "success",
  "payment_link": "https://ashtechpay.top/pay/hp-ab12cd34",
  "payment_id": "uuid-du-lien",
  "slug": "hp-ab12cd34",
  "is_fixed_amount": true,
  "amount": 5000,
  "currency": "XAF",
  "allowed_countries": null,
  "expires_at": "2026-03-15T15:30:00.000Z"
}
```

---

## 3. Prix fixe — tu définis le montant

Le montant est défini à la création. Le client voit le montant sur la page et ne peut pas le modifier. Idéal pour les produits, abonnements, factures.

```javascript
fetch("https://ashtechpay.top/api/v1/hosted-payment/create", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.HP_LIVE_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    currency: "XAF",
    amount: 5000,
    description: "Abonnement mensuel",
    is_fixed_amount: true,
    // notify_url definie dans vos parametres — recuperee automatiquement
  }),
})
```

---

## 4. Prix libre — le client choisit le montant

La page de paiement affiche un champ de saisie pour le montant. Le client entre ce qu'il veut payer. Idéal pour les dons, pourboires, paiements à montant variable.

```javascript
fetch("https://ashtechpay.top/api/v1/hosted-payment/create", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.HP_LIVE_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    currency: "XOF",
    description: "Don libre",
    is_fixed_amount: false, // <- le client saisit son montant
    // pas besoin de "amount"
  }),
})
```

> En mode prix libre, le montant réel payé par le client est disponible dans `GET /api/v1/hosted-payment/:payment_id` une fois le statut passé à `success`, dans le champ `amount`.

---

## 5. Filtrer les pays affichés

Par défaut, tous les pays actifs sont disponibles. Tu peux restreindre à un sous-ensemble en passant leurs codes ISO dans `allowed_countries`.

```javascript
body: JSON.stringify({
  currency: "XAF",
  amount: 10000,
  description: "Achat produit",
  allowed_countries: ["CM", "SN"], // uniquement Cameroun + Senegal
})
```

### 22 pays disponibles

| Code ISO | Pays | Wallet crédité | Opérateurs |
|---|---|---|---|
| CM | Cameroun | XAF | MTN, Orange |
| SN | Sénégal | XOF | Orange, Wave, Free |
| CI | Côte d'Ivoire | XOF | Orange, MTN, Wave |
| BJ | Bénin | XOF | MTN, Moov |
| BF | Burkina Faso | XOF | Orange, Moov, Coris |
| ML | Mali | XOF | Orange, Moov |
| TG | Togo | XOF | Flooz, Tmoney |
| NE | Niger | XOF | Orange, Airtel |
| GW | Guinée-Bissau | XOF | MTN |
| GN | Guinée | GNF | Orange, MTN |
| CD | Congo RDC | CDF | Airtel, Orange |
| GA | Gabon | XAF | Airtel, Moov |
| CG | Congo | XAF | Airtel, MTN |
| CF | Centrafrique | XAF | Orange |
| TD | Tchad | XAF | Airtel, Moov |
| RW | Rwanda | RWF | MTN, Airtel |
| GH | Ghana | GHS | MTN, Vodafone, Airtel |
| NG | Nigeria | NGN | MTN, Airtel |
| KE | Kenya | KES | M-Pesa |
| TZ | Tanzanie | TZS | Vodacom, Airtel, Tigo |
| UG | Ouganda | UGX | MTN, Airtel |
| GM | Gambie | GMD | Afrimoney, QMoney |

> Si `allowed_countries` est absent ou vide, tous les pays actifs sont disponibles. L'administrateur contrôle quels pays et opérateurs sont actifs — tout changement s'applique automatiquement sans modifier votre code.

---

## 6. Vérifier le statut d'un paiement

`GET /api/v1/hosted-payment/:payment_id`

```json
{
  "payment_id": "uuid-du-lien",
  "slug": "hp-ab12cd34",
  "is_fixed_amount": true,
  "amount": 5000,
  "currency": "XAF",
  "description": "Abonnement Premium",
  "allowed_countries": ["CM"],
  "status": "success",
  "paid_at": "2026-03-15T15:12:34.000Z",
  "created_at": "2026-03-15T15:00:00.000Z",
  "expires_at": "2026-03-15T15:30:00.000Z"
}
```

| Statut | Signification | Action |
|---|---|---|
| `pending` | Lien créé — le client n'a pas encore payé | Continuer à poller (toutes les 5s) |
| `processing` | Le client a initié le paiement | Continuer à poller |
| `success` | Paiement confirmé — wallet crédité | Livrer le produit / service |
| `failed` | Paiement échoué ou refusé | Notifier le client |
| `expired` | 30 min dépassées sans paiement | Créer un nouveau lien |

> ⚠️ **Important** — Ne livre jamais avant d'avoir vérifié `status === "success"`. Le statut `processing` signifie que le paiement est initié mais pas encore confirmé par l'opérateur.

---

## 7. Créditement automatique du wallet

Dès que l'opérateur Mobile Money confirme le paiement, Ashtech Pay crédite automatiquement ton wallet marchand dans la devise du pays du client. Aucune action requise.

### Étapes du créditement

1. Le client confirme le paiement sur son téléphone (USSD / OTP / Wave)
2. L'opérateur Mobile Money notifie Ashtech Pay
3. La transaction est enregistrée comme `completed`
4. Ton wallet marchand est crédité dans la devise du pays (frais déduits)
5. Le statut passe à `success` — tu peux livrer

### Wallet crédité par pays

| Pays du client | Wallet marchand crédité |
|---|---|
| Bénin | XOF |
| Sénégal | XOF |
| Côte d'Ivoire | XOF |
| Burkina Faso | XOF |
| Mali | XOF |
| Togo | XOF |
| Niger | XOF |
| Guinée-Bissau | XOF |
| Cameroun | XAF |
| Gabon | XAF |
| Congo | XAF |
| Centrafrique | XAF |
| Tchad | XAF |
| Guinée | GNF |
| Congo RDC | CDF |
| Rwanda | RWF |
| Ghana | GHS |
| Nigeria | NGN |
| Kenya | KES |
| Tanzanie | TZS |
| Ouganda | UGX |
| Gambie | GMD |

> Chaque pays crédite un wallet séparé dans ta balance. Tu peux ensuite convertir ces wallets en XOF, XAF ou toute autre devise depuis ton tableau de bord → Wallets.

---

## 8. Webhook — notification automatique

Configure ta Webhook URL une seule fois dans tes paramètres **API Keys**. Ashtech Pay la récupère automatiquement à chaque paiement Hosted Page.

> Tu peux aussi passer un `notify_url` spécifique lors de la création d'un lien — il remplacera l'URL configurée par défaut pour ce lien uniquement. Le webhook est envoyé depuis nos serveurs vers ton serveur — l'URL doit être publiquement accessible (pas localhost).

### Payload reçu (POST → votre serveur)

```json
{
  "event": "payment.completed",
  "transaction_id": "uuid-de-la-txn",
  "reference": "ASHPAY-DEP-XXXXXXXXXX",
  "status": "completed",
  "amount": 4750,
  "total_amount": 5000,
  "currency": "XAF",
  "type": "deposit",
  "phone": "656123456",
  "timestamp": "2026-03-16T03:00:00.000Z"
}
```

> `event` peut être `"payment.completed"` ou `"payment.failed"` · `amount` = montant net crédité sur ton wallet (frais déduits) · `total_amount` = montant brut payé par le client

### Exemple de récepteur webhook (Node.js / Express)

```javascript
app.post("/webhooks/ashtechpay", express.json(), (req, res) => {
  // Toujours répondre 200 d'abord, traiter ensuite
  res.sendStatus(200);

  const { event, transaction_id, reference, amount, total_amount, currency } = req.body;

  if (event === "payment.completed") {
    // amount = montant net crédité sur votre wallet (après frais)
    // total_amount = montant brut payé par le client
    console.log(`Paiement recu : ${amount} ${currency}`);
    // crediter le compte client, livrer la commande...
  }
  if (event === "payment.failed") {
    console.log(`Paiement echoue | ref: ${reference}`);
    // annuler la commande, notifier le client...
  }
});
```

> ⚠️ Réponds toujours HTTP 200 immédiatement — même si une erreur survient côté serveur. Si ton serveur répond autre chose, le webhook ne sera pas renvoyé.

---

## 9. Exemples de code

### Node.js — créer et surveiller un lien de paiement

```javascript
const HP_KEY = process.env.HP_LIVE_KEY;

// Creer un lien de paiement
async function createLink({ amount, currency, description, countries }) {
  const res = await fetch("https://ashtechpay.top/api/v1/hosted-payment/create", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${HP_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      currency, amount, description,
      is_fixed_amount: !!amount,
      allowed_countries: countries ?? null,
    }),
  });
  return res.json();
  // { payment_link, payment_id, expires_at, ... }
}

// Verifier le statut
async function checkStatus(paymentId) {
  const res = await fetch(
    `https://ashtechpay.top/api/v1/hosted-payment/${paymentId}`,
    { headers: { "Authorization": `Bearer ${HP_KEY}` } }
  );
  return res.json();
}

// Polling toutes les 5 secondes
const link = await createLink({ amount: 5000, currency: "XAF", description: "Commande #123" });
const interval = setInterval(async () => {
  const { status } = await checkStatus(link.payment_id);
  if (status === "success") {
    clearInterval(interval);
    console.log("Paiement confirme — livrer le produit");
  } else if (status === "failed" || status === "expired") {
    clearInterval(interval);
    console.log("Paiement non abouti :", status);
  }
}, 5000);
```

### PHP

```php
<?php
$hpKey = getenv("HP_LIVE_KEY");

function createPaymentLink($currency, $amount, $description, $countries = null) {
    global $hpKey;
    $payload = array_filter([
        "currency" => $currency,
        "amount" => $amount,
        "description" => $description,
        "is_fixed_amount" => !empty($amount),
        "allowed_countries" => $countries,
    ]);

    $ch = curl_init("https://ashtechpay.top/api/v1/hosted-payment/create");
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            "Authorization: Bearer $hpKey",
            "Content-Type: application/json",
        ],
        CURLOPT_POSTFIELDS => json_encode($payload),
    ]);

    $result = json_decode(curl_exec($ch), true);
    curl_close($ch);
    return $result;
}

$link = createPaymentLink("XAF", 10000, "Facture #456", ["CM"]);
header("Location: " . $link["payment_link"]);
```

### cURL

```bash
# Prix fixe — Cameroun uniquement
curl -X POST https://ashtechpay.top/api/v1/hosted-payment/create \
  -H "Authorization: Bearer hp_live_xxxxxxxxxxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"currency":"XAF","amount":5000,"description":"Commande","allowed_countries":["CM"]}'

# Prix libre — tous les pays
curl -X POST https://ashtechpay.top/api/v1/hosted-payment/create \
  -H "Authorization: Bearer hp_live_xxxxxxxxxxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"currency":"XOF","description":"Don","is_fixed_amount":false}'

# Verifier le statut
curl https://ashtechpay.top/api/v1/hosted-payment/UUID_DU_LIEN \
  -H "Authorization: Bearer hp_live_xxxxxxxxxxxxxxxxxxxxxxxx"
```

---

## Support

Pour toute question technique non résolue par cette documentation, contactez notre équipe via le support intégré à l'application, ou par email : **support@ashtechpay.top**

- Website: [ashtechpay.top](https://ashtechpay.top)
- API Base URL: `https://ashtechpay.top`
