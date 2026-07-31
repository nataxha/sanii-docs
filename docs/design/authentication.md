# Authentification — Documentation

## Vue d'ensemble du flux

```
Première connexion                      Connexions suivantes
─────────────────                       ────────────────────
App démarre                             App démarre
    │                                       │
AuthGate (écoute onAuthStateChange)     AuthGate (écoute onAuthStateChange)
    │                                       │
Pas de session → /login             Session trouvée → /home
    │                                   (automatique, sans OTP)
Utilisateur entre son numéro
    │
Supabase envoie SMS via Twilio
    │
Utilisateur entre le code OTP
    │
Supabase vérifie le code
    │
Session créée → /home
```

---

## Étapes détaillées

### 1. Démarrage de l'app — `AuthGate`

`AuthGate` écoute le stream `onAuthStateChange` de Supabase.

- Ce stream émet **immédiatement** l'état actuel de la session au démarrage
- Si une session valide existe en local → redirige vers `/`
- Si aucune session → redirige vers `/login`

### 2. Saisie du numéro — `PhoneInputScreen`

```dart
await Supabase.instance.client.auth.signInWithOtp(phone: phone);
```

- Supabase appelle l'API Twilio
- Twilio envoie un SMS avec un code OTP à 6 chiffres
- L'utilisateur est redirigé vers l'écran de vérification

### 3. Vérification du code — `OtpVerificationScreen`

```dart
await Supabase.instance.client.auth.verifyOTP(
  phone: phone,
  token: code,
  type: OtpType.sms,
);
```

- Supabase vérifie le code OTP
- Si correct → Supabase crée un utilisateur (si nouveau) ou reconnecte (si existant)
- Une **session** est générée avec :
  - `access_token` (JWT valide 1 heure)
  - `refresh_token` (valide longtemps, pour renouveler l'access_token)
- `onAuthStateChange` émet la nouvelle session → `AuthGate` redirige vers `/`

---

## Ce qui est sauvegardé localement

`supabase_flutter` utilise **`SharedPreferences`** (Android/iOS) pour persister automatiquement :

| Donnée | Description |
|--------|-------------|
| `access_token` | JWT envoyé à chaque requête Supabase |
| `refresh_token` | Permet de renouveler l'access_token expiré |
| `user` | Informations de l'utilisateur (id, phone, metadata) |
| `expires_at` | Timestamp d'expiration de l'access_token |

**Aucune action manuelle nécessaire** — c'est géré automatiquement par le SDK.

---

## Est-ce qu'on demande toujours à l'utilisateur de se connecter ?

**Non.** Grâce au `refresh_token` :

- L'`access_token` expire après **1 heure**
- Mais Supabase le **renouvelle automatiquement** en arrière-plan avec le `refresh_token`
- Le `refresh_token` expire après **plusieurs semaines** (configurable dans le dashboard)
- Tant que le `refresh_token` est valide, l'utilisateur reste connecté sans intervention

L'utilisateur est redemandé de se connecter uniquement si :
- Il se déconnecte manuellement (`supabase.auth.signOut()`)
- Le `refresh_token` expire (session très ancienne ou révoquée)
- Il réinstalle l'app (stockage effacé)

---

## Est-ce une bonne pratique ?

**Oui**, c'est le standard de l'industrie (OAuth 2.0).

| Aspect | Notre implémentation | Évaluation |
|--------|---------------------|------------|
| Stockage des tokens | `SharedPreferences` via SDK Supabase | ✅ Standard |
| Renouvellement automatique | Géré par `supabase_flutter` | ✅ Transparent |
| Expiration de session | Access token 1h, refresh token configurable | ✅ Sécurisé |
| Aucun mot de passe stocké | OTP uniquement, pas de credential persisté | ✅ Bonne pratique |
| JWT signé par Supabase | Vérifié par PostgREST pour chaque requête DB | ✅ Sécurisé |

### Ce qu'on pourrait améliorer (optionnel)

- Utiliser `flutter_secure_storage` à la place de `SharedPreferences` pour chiffrer les tokens au repos (utile si l'appareil est rooté)
- Configurer la durée du `refresh_token` dans **Dashboard → Authentication → Settings → JWT expiry**
- Activer le **logout automatique sur tous les appareils** en cas de compromission via `supabase.auth.signOut(scope: SignOutScope.global)`
