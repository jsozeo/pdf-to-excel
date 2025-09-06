# API Documentation - bank2xls

## Base URL
```
https://api.convertir-releve-compte-pdf-excel.com
```

## Authentification
L'API utilise un système de sessions avec cookies :
- **Session anonyme** : Cookie `pg1_session` (7 jours)
- **Session authentifiée** : Cookies `pg1_session` + `pg1_auth` (30 jours)

## Headers requis
```http
Content-Type: application/json
```

## Codes d'erreur
| Code | Message | Description |
|------|---------|-------------|
| `INVALID_EMAIL` | Adresse email invalide | Format email incorrect |
| `OTP_INVALID` | Code OTP invalide | Code OTP incorrect |
| `OTP_EXPIRED` | Code OTP expiré | Code OTP expiré (10 min) |
| `OTP_CONSUMED` | Code OTP déjà utilisé | Code OTP déjà utilisé |
| `OTP_RATE_LIMITED` | Trop de tentatives | Rate limit dépassé |
| `DAILY_QUOTA_EXCEEDED` | Quota quotidien gratuit dépassé | Crédit gratuit déjà utilisé |
| `INSUFFICIENT_FUNDS` | Crédits insuffisants | Pas assez de crédits |
| `PLAN_UNKNOWN` | Plan inconnu | Plan de facturation invalide |
| `STRIPE_ERROR` | Erreur de paiement | Erreur Stripe |
| `NOT_FOUND` | Ressource non trouvée | Endpoint inexistant |
| `FORBIDDEN` | Accès refusé | Authentification requise |
| `INTERNAL_ERROR` | Erreur interne | Erreur serveur |

---

## Endpoints

### 1. Health Check
**GET** `/api/health`

Vérifie l'état des services.

**Réponse :**
```json
{
  "status": "ok",
  "time": "2025-01-27T10:30:00.000Z",
  "dynamo": "ok",
  "s3": "ok",
  "stripe": "ok",
  "ses": "ok"
}
```

---

### 2. Authentification

#### 2.1 Demander un code OTP
**POST** `/api/auth/otp/request`

**Body :**
```json
{
  "email": "user@example.com"
}
```

**Réponse :**
```json
{
  "status": "OTP_SENT",
  "message": "Code envoyé par email"
}
```

**Rate Limits :**
- 5 OTP par email/heure
- 20 OTP par IP/jour

#### 2.2 Vérifier le code OTP
**POST** `/api/auth/otp/verify`

**Body :**
```json
{
  "email": "user@example.com",
  "otp": "123456"
}
```

**Réponse :**
```json
{
  "status": "AUTHENTICATED",
  "userId": "user_123",
  "isNewUser": false
}
```

**Cookies :** Définit `pg1_session` et `pg1_auth`

#### 2.3 Déconnexion
**POST** `/api/auth/logout`

**Réponse :**
```json
{
  "status": "LOGGED_OUT"
}
```

**Cookies :** Supprime les cookies de session

---

### 3. Crédits

#### 3.1 Obtenir le solde de crédits
**GET** `/api/credits/balance`

**Réponse anonyme :**
```json
{
  "mode": "anonymous",
  "freeDaily": {
    "limit": 1,
    "usedToday": 0,
    "remaining": 1,
    "resetAt": "2025-01-28T00:00:00.000Z"
  }
}
```

**Réponse authentifiée :**
```json
{
  "mode": "authenticated",
  "paidCredits": {
    "balance": 50,
    "lastUpdate": "2025-01-27T10:30:00.000Z"
  },
  "subscription": {
    "status": "active",
    "planId": "monthly_100",
    "nextInvoiceAt": "2025-02-27T00:00:00.000Z"
  },
  "rollover": true
}
```

#### 3.2 Consommer des crédits
**POST** `/api/credits/consume`

**Body :**
```json
{
  "reason": "bank2xls_conversion",
  "amount": 1,
  "requestId": "req_123"
}
```

**Réponse anonyme :**
```json
{
  "debited": 1,
  "type": "free-daily",
  "date": "2025-01-27",
  "remainingToday": 0
}
```

**Réponse authentifiée :**
```json
{
  "debited": 1,
  "newBalance": 49,
  "entryId": "entry_123"
}
```

---

### 4. Facturation

#### 4.1 Créer une session de checkout
**POST** `/api/billing/checkout`

**Authentification :** Requise

**Body :**
```json
{
  "planId": "monthly_100"
}
```

**Réponse :**
```json
{
  "checkoutUrl": "https://checkout.stripe.com/pay/cs_123..."
}
```

#### 4.2 Accéder au portail client
**POST** `/api/billing/portal`

**Authentification :** Requise

**Réponse :**
```json
{
  "portalUrl": "https://billing.stripe.com/p/session_123..."
}
```

#### 4.3 Obtenir les factures
**GET** `/api/billing/invoices`

**Authentification :** Requise

**Réponse :**
```json
{
  "invoices": [
    {
      "id": "in_123",
      "amount": 1999,
      "currency": "eur",
      "status": "paid",
      "created": "2025-01-27T10:30:00.000Z",
      "invoiceUrl": "https://invoice.stripe.com/i/acct_123/in_123"
    }
  ]
}
```

---

### 5. Historique des conversions

#### 5.1 Obtenir l'historique
**GET** `/api/conversions/history`

**Authentification :** Requise

**Query Parameters :**
- `limit` (optionnel) : Nombre de résultats (défaut: 10)

**Réponse :**
```json
{
  "conversions": [
    {
      "conversionId": "conv_123",
      "userId": "user_123",
      "sessionId": "session_123",
      "type": "bank2xls",
      "cost": 1,
      "status": "success",
      "createdAt": "2025-01-27T10:30:00.000Z",
      "metadata": {
        "fileName": "releve.pdf",
        "fileSize": 1024000,
        "processingTime": 2.5
      }
    }
  ]
}
```

---

### 6. Pages HTML

#### 6.1 Page d'accueil
**GET** `/`

Retourne la page d'accueil HTML.

#### 6.2 Dashboard
**GET** `/app`

**Authentification :** Requise

Retourne la page dashboard HTML.

#### 6.3 Facturation
**GET** `/billing`

**Authentification :** Requise

Retourne la page de facturation HTML.

---

### 7. Webhooks

#### 7.1 Webhook Stripe
**POST** `/api/webhooks/stripe`

**Headers :**
```http
Stripe-Signature: t=1234567890,v1=signature...
```

**Body :** Événement Stripe (voir documentation Stripe)

**Réponse :**
```json
{
  "status": "processed"
}
```

---

## Gestion des erreurs

### Format des erreurs
```json
{
  "error": {
    "code": "OTP_RATE_LIMITED",
    "message": "Trop de tentatives. Veuillez patienter avant de redemander un code"
  }
}
```

### Codes de statut HTTP
- `200` : Succès
- `400` : Erreur de validation
- `401` : Non authentifié
- `402` : Paiement requis
- `403` : Accès refusé
- `404` : Non trouvé
- `409` : Conflit (idempotency)
- `429` : Trop de requêtes
- `500` : Erreur serveur

---

## CORS

L'API supporte CORS avec les headers suivants :
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET,POST,OPTIONS`
- `Access-Control-Allow-Headers: Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token,X-Request-Id,X-Idempotency-Key`
- `Access-Control-Allow-Credentials: true`

---

## Rate Limiting

- **OTP par email :** 5/heure
- **OTP par IP :** 20/jour
- **Headers de réponse :**
  - `X-RateLimit-Limit`
  - `X-RateLimit-Remaining`
  - `X-RateLimit-Reset`

---

## Exemples d'utilisation

### Connexion complète
```javascript
// 1. Demander OTP
const otpResponse = await fetch('/api/auth/otp/request', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email: 'user@example.com' })
});

// 2. Vérifier OTP
const verifyResponse = await fetch('/api/auth/otp/verify', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email: 'user@example.com', otp: '123456' })
});

// 3. Obtenir le solde
const balanceResponse = await fetch('/api/credits/balance', {
  credentials: 'include' // Important pour les cookies
});
```

### Utilisation anonyme
```javascript
// 1. Obtenir le solde (crée une session anonyme)
const balanceResponse = await fetch('/api/credits/balance', {
  credentials: 'include'
});

// 2. Consommer un crédit gratuit
const consumeResponse = await fetch('/api/credits/consume', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({ reason: 'bank2xls_conversion' })
});
```

### Gestion des erreurs
```javascript
async function apiCall(url, options = {}) {
  try {
    const response = await fetch(url, {
      credentials: 'include',
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      },
      ...options
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.error?.message || 'Erreur API');
    }

    return await response.json();
  } catch (error) {
    console.error('API Error:', error);
    throw error;
  }
}

// Utilisation
try {
  const balance = await apiCall('/api/credits/balance');
  console.log('Solde:', balance);
} catch (error) {
  console.error('Erreur:', error.message);
}
```

### Vérification de l'état d'authentification
```javascript
async function checkAuthStatus() {
  try {
    const response = await fetch('/api/credits/balance', {
      credentials: 'include'
    });
    
    if (response.ok) {
      const data = await response.json();
      return {
        isAuthenticated: data.mode === 'authenticated',
        isAnonymous: data.mode === 'anonymous',
        data: data
      };
    }
  } catch (error) {
    console.error('Erreur de vérification auth:', error);
  }
  
  return { isAuthenticated: false, isAnonymous: false };
}
```

---

## Notes importantes

1. **Cookies** : Toujours inclure `credentials: 'include'` dans les requêtes fetch
2. **Rate Limiting** : Respecter les limites pour éviter les erreurs 429
3. **Gestion d'erreurs** : Toujours vérifier le statut HTTP et parser les erreurs JSON
4. **Sessions** : Les sessions anonymes sont créées automatiquement lors du premier appel
5. **CORS** : L'API supporte les requêtes cross-origin avec credentials

---

## Changelog

### Version 1.0.0
- Authentification OTP par email
- Gestion des crédits (anonymes et payants)
- Intégration Stripe pour la facturation
- Système de sessions avec cookies
- Rate limiting pour les OTP
- Support CORS complet
