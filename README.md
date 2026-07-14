# Ashtech Pay — Direct API SDK Documentation

**Version:** v1 (Mai 2026)
**Mobile Money — 16 pays africains**

Initiez des paiements Mobile Money directement depuis votre serveur, sans redirection. Gérez les flux USSD Push, OTP SMS, OTP USSD et Wave en 16 pays africains.

| | |
|---|---|
| **Endpoint Collect** | `POST /v1/collect` |
| **Authentification** | `Bearer YOUR_API_KEY` |
| **Base URL** | `https://ashtechpay.top` |
| **Version** | v1 — Mai 2026 |
| **Contact** | support@ashtechpay.top |

## Sommaire

1. [Introduction](#1-introduction)
2. [Authentification](#2-authentification)
3. [GET /v1/countries — Pays et opérateurs](#3-pays-et-opérateurs--get-v1countries)
4. [POST /v1/collect — Initier un paiement](#4-initier-un-paiement--post-v1collect)
5. [Flux de paiement](#5-flux-de-paiement)
6. [GET /v1/transaction/:id — Statut d'une transaction](#6-statut-dune-transaction--get-v1transactionid)
7. [GET /v1/fees — Grille tarifaire en temps réel](#7-grille-tarifaire-en-temps-réel--get-v1fees)
8. [Webhooks](#8-webhooks)
9. [Codes d'erreur](#9-codes-derreur)

---

## 1. Introduction

L'Ashtech Pay API unifie plusieurs passerelles de paiement africaines en une seule interface REST. Initiez des paiements Mobile Money dans 22+ pays africains sans redirection. Le routage entre les opérateurs est automatique — vous n'avez pas à choisir le fournisseur.

| Caractéristique | Détail |
|---|---|
| Base URL | `https://ashtechpay.top` |
| Format | JSON uniquement — `Content-Type: application/json` |
| Authentification | Bearer token dans l'en-tête `Authorization` |
| Protocole | HTTPS obligatoire |
| Versioning | Préfixe `/v1/` dans tous les endpoints |

---

## 2. Authentification

Toutes les requêtes doivent inclure votre clé API dans l'en-tête HTTP `Authorization`.

```http
Authorization: Bearer YOUR_API_KEY
```

> ⚠️ Utilisez votre clé API uniquement depuis votre serveur (Node.js, Python, PHP…). Ne l'incluez jamais dans du code côté navigateur ou application mobile.

Exemple d'appel authentifié (Node.js) :

```javascript
const response = await fetch("https://ashtechpay.top/v1/collect", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ /* ... */ })
});
```

---

## 3. Pays et opérateurs — GET /v1/countries

Retourne la liste complète des pays actifs et leurs opérateurs Mobile Money disponibles. Cette liste est gérée par l'administrateur — tout ajout ou retrait de pays/opérateur est immédiatement visible via cet endpoint.

```javascript
fetch("https://ashtechpay.top/v1/countries", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
})
```

```bash
curl https://ashtechpay.top/v1/countries \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**Réponse**

```json
[
  {
    "code": "CM",
    "name": "Cameroun",
    "currency": "XAF",
    "operators": ["MTN Mobile Money", "Orange Money"]
  },
  {
    "code": "SN",
    "name": "Senegal",
    "currency": "XOF",
    "operators": ["Free Money", "Orange Money", "Wave"]
  }
]
```

### Pays disponibles (16)

| Pays | Code | Devise | Opérateurs |
|---|---|---|---|
| Bénin | BJ | XOF | Moov Money, MTN Mobile Money |
| Burkina Faso | BF | XOF | Moov Money, Orange Money (OTP) |
| Cameroun | CM | XAF | MTN Mobile Money, Orange Money |
| Centrafrique | CF | XAF | Orange Money (OTP) |
| Congo | CG | XAF | Airtel Money, MTN Mobile Money |
| Côte d'Ivoire | CI | XOF | Moov Money, MTN, Orange (OTP), Wave |
| Gabon | GA | XAF | Airtel Money, Moov Money |
| Guinée Conakry | GN | GNF | MTN Mobile Money, Orange Money |
| Guinée équat. | GQ | XAF | Orange Money (OTP) |
| Guinée-Bissau | GW | XOF | Orange Money (OTP) |
| Mali | ML | XOF | Moov Money, Orange Money (OTP) |
| Niger | NE | XOF | Airtel Money |
| RD Congo | CD | CDF | Afrimoney, Airtel, Orange (OTP), Vodacom M-Pesa |
| Sénégal | SN | XOF | Free Money, Orange Money (OTP), Wave |
| Tchad | TD | XAF | Airtel Money, Moov Money |
| Togo | TG | XOF | Flooz (Moov), T-Money |

**Légende :** `(OTP USSD)` = code à composer pour recevoir l'OTP · `(OTP SMS)` = SMS automatique · `Wave` = lien de paiement Wave

---

## 4. Initier un paiement — POST /v1/collect

Initie un paiement Mobile Money. Le client reçoit une demande de validation sur son téléphone. Le routage entre fournisseurs est automatique selon le pays et l'opérateur. Les frais sont déduits automatiquement — le champ `credited_amount` est le montant net crédité.

### Corps de la requête (JSON)

| Paramètre | Type | Statut | Description |
|---|---|---|---|
| `amount` | number | Requis | Montant brut à collecter |
| `currency` | string | Requis | Devise du pays (XAF, XOF, GNF, CDF…) |
| `phone` | string | Requis | Numéro de téléphone du payeur |
| `operator` | string | Requis | Nom exact de l'opérateur (depuis `/v1/countries`) |
| `country_code` | string | Requis | Code ISO du pays (CM, SN, CI…) |
| `reference` | string | Optionnel* | Référence unique de votre commande. *Obligatoire lors du retry OTP |
| `otp` | string | Optionnel | Code OTP reçu par SMS. Doit être accompagné du champ `reference` |
| `notify_url` | string | Optionnel | URL webhook pour recevoir le résultat du paiement |

### Requête exemple

```javascript
fetch("https://ashtechpay.top/v1/collect", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    amount: 5000,
    currency: "XAF",
    phone: "670000000",
    operator: "MTN Mobile Money",
    country_code: "CM",
    reference: "ORDER-001",
    notify_url: "https://monsite.com/webhook"
  })
})
```

```bash
curl https://ashtechpay.top/v1/collect \
  -X POST \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"amount":5000,"currency":"XAF","phone":"670000000",
      "operator":"MTN Mobile Money","country_code":"CM",
      "reference":"ORDER-001",
      "notify_url":"https://monsite.com/webhook"}'
```

### Réponse 202 (succès USSD Push)

```json
{
  "transaction_id": "8f3e1c2d-...",
  "reference": "ORDER-001",
  "status": "pending",
  "amount": 5000,
  "credited_amount": 4750,
  "fee_amount": 250,
  "currency": "XAF",
  "operator": "MTN Mobile Money",
  "phone": "670000000",
  "country_code": "CM",
  "created_at": "2026-03-15T14:00:00Z"
}
```

### OTP requis — Orange CI/SN/BF/ML (USSD) et LigdiCash BF (SMS)

Orange CI/SN/BF/ML → OTP USSD : le serveur retourne un `ussd_code` à afficher au client (l'OTP s'affiche dans le menu, aucun SMS envoyé). LigdiCash BF → OTP SMS envoyé automatiquement (`ussd_code = null`). Dans les deux cas, la réponse 400 contient un champ `reference` obligatoire pour l'étape 2.

```json
// Étape 1 — Requête initiale (sans otp) → réponse 400
{
  "error": "otp_required",
  "message": "OTP requis. Un code a été envoyé par SMS.",
  "reference": "DEP-A1B2C3D4",
  "ussd_code": null
}
```

```json
// Étape 2 — Retry avec OTP reçu + reference du 400 → réponse 202
{
  "amount": 5000,
  "currency": "XOF",
  "phone": "07XXXXXXXX",
  "operator": "Orange Money",
  "country_code": "CI",
  "otp": "123456",
  "reference": "DEP-A1B2C3D4",
  "notify_url": "https://monsite.com/webhook"
}
```

---

## 5. Flux de paiement

Selon le pays et l'opérateur, l'API utilise automatiquement l'un des 4 flux ci-dessous. Votre code doit gérer chacun différemment car la réponse et les étapes varient.

| Flux | Opérateurs concernés | Réponse initiale | Action requise |
|---|---|---|---|
| **USSD Push** | MTN, Moov, Airtel, Orange CM, Free SN, T-Money, Flooz, M-Pesa | `202 pending` | Attendre le webhook. Le client valide sur son téléphone. |
| **OTP USSD** | Orange CI (`#144*82#`), SN (`#144*391#`), BF (`*144*4*6*montant#`) | `400 otp_required`, `reference: "DEP-..."`, `ussd_code: "#144*82#"` | L'API envoie l'OTP par SMS/USSD. Relancer avec `otp` + `reference`. |
| **OTP SMS** | LigdiCash BF (wallet) | `400 otp_required`, `reference: "DEP-..."`, `ussd_code: null` | SMS envoyé automatiquement. Relancer avec `otp` + `reference`. |
| **Wave** | Wave CI, Wave SN | `202 pending`, `flow: "wave"`, `wave_url: ...` | Afficher le `wave_url` en bouton ou QR code. Le client ouvre le lien. |

### Détection du flux dans votre code

```javascript
async function collectPayment(params) {
  const res = await fetch("https://ashtechpay.top/v1/collect", {
    method: "POST",
    headers: { "Authorization": "Bearer YOUR_API_KEY", "Content-Type": "application/json" },
    body: JSON.stringify(params)
  });
  const data = await res.json();

  if (res.status === 202 && data.flow === "wave") {
    // Flux Wave : afficher data.wave_url
    return { type: "wave", waveUrl: data.wave_url, transactionId: data.transaction_id };
  }
  if (res.status === 202) {
    // Flux USSD Push : attendre webhook
    return { type: "ussd_push", transactionId: data.transaction_id };
  }
  if (res.status === 400 && data.error === "otp_required") {
    // Stocker data.reference — obligatoire pour le retry OTP
    if (data.ussd_code) {
      // OTP USSD (Orange CI, SN, BF) : afficher le code à composer
      // CI=#144*82# SN=#144*391# BF=*144*4*6*montant#
      return { type: "otp_ussd", ussdCode: data.ussd_code, reference: data.reference };
    } else {
      // OTP SMS (Orange CI, SN, ML, LigdiCash BF…) : SMS envoyé automatiquement
      return { type: "otp_sms", reference: data.reference };
    }
  }
  throw new Error(data.message);
}
```

---

## 6. Statut d'une transaction — GET /v1/transaction/:id

Consultez le statut d'une transaction à tout moment via le `transaction_id` retourné lors de l'initiation. Vous pouvez utiliser ce endpoint en complément du webhook.

```javascript
fetch("https://ashtechpay.top/v1/transaction/8f3e1c2d-...", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
})
```

```bash
curl https://ashtechpay.top/v1/transaction/8f3e1c2d-... \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**Réponse**

```json
{
  "transaction_id": "8f3e1c2d-...",
  "reference": "ORDER-001",
  "status": "success",
  "amount": 5000,
  "credited_amount": 4750,
  "fee_amount": 250,
  "currency": "XAF",
  "phone": "670000000",
  "created_at": "2026-03-15T14:00:00Z",
  "confirmed_at": "2026-03-15T14:02:17Z"
}
```

| Statut | Description | Final ? |
|---|---|---|
| `pending` | En attente de confirmation de l'opérateur | Non |
| `success` | Paiement confirmé — compte marchand crédité | Oui |
| `failed` | Paiement refusé, expiré ou annulé | Oui |

---

## 7. Grille tarifaire en temps réel — GET /v1/fees

Retourne la grille tarifaire en vigueur pour chaque pays actif. Les frais sont configurés par l'administrateur et peuvent changer à tout moment. Consultez cet endpoint pour calculer le montant net avant d'appeler `/v1/collect`.

```javascript
fetch("https://ashtechpay.top/v1/fees", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
})
```

```bash
curl https://ashtechpay.top/v1/fees \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**Réponse**

```json
[
  {
    "country_code": "CM",
    "country_name": "Cameroun",
    "currency": "XAF",
    "deposit_fee_pct": 3.5,
    "withdrawal_fee_pct": 1.5,
    "transfer_fee_pct": 1.0,
    "total_fee_pct": 5.5
  }
]
```

### Exemple — calculer le montant net avant d'appeler /v1/collect

```javascript
// Récupérer les frais en cache (une fois au démarrage ou toutes les heures)
const fees = await fetch("https://ashtechpay.top/v1/fees", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
}).then(r => r.json());

// Calculer le montant net crédité sur votre compte
function computeNet(grossAmount, countryCode) {
  const fee = fees.find(f => f.country_code === countryCode);
  if (!fee) return grossAmount;
  const feeAmount = Math.round(grossAmount * fee.total_fee_pct / 100);
  return {
    gross: grossAmount,
    fee: feeAmount,
    net: grossAmount - feeAmount,
    fee_pct: fee.total_fee_pct,
  };
}

// Exemple :
console.log(computeNet(10000, "CM"));
// -> { gross: 10000, fee: 550, net: 9450, fee_pct: 5.5 }
```

---

## 8. Webhooks

Quand une transaction atteint un état final, Ashtech Pay envoie automatiquement une requête `POST` à la `notify_url` passée dans votre appel à `/v1/collect`. Le champ `amount` correspond au montant net après frais, et `total_amount` au montant brut collecté.

### Payload — paiement réussi

```json
{
  "event": "payment.completed",
  "transaction_id": "8f3e1c2d-...",
  "reference": "ORDER-001",
  "status": "completed",
  "amount": 4750,
  "total_amount": 5000,
  "currency": "XAF",
  "type": "deposit",
  "phone": "670000000",
  "timestamp": "2026-03-15T14:02:17.000Z"
}
```

> `amount` = net après frais · `total_amount` = brut collecté

### Événements disponibles

| Événement | Déclencheur |
|---|---|
| `payment.completed` | Paiement (dépôt) confirmé avec succès |
| `payment.failed` | Paiement refusé, expiré ou annulé |
| `payout.completed` | Retrait ou virement sortant confirmé |
| `payout.failed` | Retrait ou virement échoué |

### Handler — Node.js / Express

```javascript
app.post("/webhook", express.json(), async (req, res) => {
  // Toujours répondre 200 en premier
  res.status(200).json({ received: true });

  const { event, transaction_id, reference, amount, currency } = req.body;

  if (event === "payment.completed") {
    // amount = montant net (après frais)
    await markOrderAsPaid(reference, { transactionId: transaction_id, amount, currency });
  }
  if (event === "payment.failed") {
    await cancelOrder(reference);
  }
  if (event === "payout.completed") {
    await markPayoutDone(reference, { transactionId: transaction_id, amount, currency });
  }
  if (event === "payout.failed") {
    await markPayoutFailed(reference);
  }
});
```

> **Bonnes pratiques :** Répondez toujours HTTP 200 immédiatement. Traitez la logique métier après avoir répondu 200 (asynchrone). Vérifiez le `transaction_id` dans votre base pour éviter les doublons. Votre `notify_url` doit être une URL HTTPS publique (pas localhost).

---

## 9. Codes d'erreur

En cas d'erreur, l'API retourne un objet JSON avec les champs `error` et `message`.

```json
{
  "error": "bad_request",
  "message": "Champs requis : amount, currency, phone, operator, country_code"
}
```

| HTTP | Erreur | Signification |
|---|---|---|
| 400 | `bad_request` | Paramètre manquant ou format invalide |
| 400 | `otp_required` | OTP requis — conservez le champ `reference` de la réponse, obligatoire pour le retry |
| 400 | `missing_reference` | Confirmation OTP sans le champ `reference` — utilisez la valeur reçue à l'étape 1 |
| 400 | `otp_expired` | Session OTP expirée (15 min) ou introuvable — relancez sans OTP |
| 401 | `unauthorized` | Clé API manquante, invalide ou révoquée |
| 403 | `forbidden` | Cette transaction n'appartient pas à votre compte |
| 404 | `not_found` | Transaction introuvable |
| 422 | `unprocessable` | Pays ou opérateur non supporté / devise incorrecte |
| 429 | `rate_limited` | Trop de requêtes — ralentissez |
| 502 | `gateway_error` | Le réseau de l'opérateur a rejeté le paiement |
| 500 | `server_error` | Erreur interne — réessayez |

---

## Support

Pour toute question technique non résolue par cette documentation, contactez notre équipe via le support intégré à l'application, ou par email : **support@ashtechpay.top**

- Website: [ashtechpay.top](https://ashtechpay.top)
- API Base URL: `https://ashtechpay.top`
