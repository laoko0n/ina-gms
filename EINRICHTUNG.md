# INA – Einrichtungsanleitung
**Individueller Nachmittag · GMS St. Wendel**

---

## Schritt 1 · Firebase-Projekt anlegen

1. Gehe zu [console.firebase.google.com](https://console.firebase.google.com)
2. Klicke **„Projekt hinzufügen"** → Namen eingeben (z. B. `ina-gms-stwend`)
3. Google Analytics kann deaktiviert bleiben → **Projekt erstellen**

---

## Schritt 2 · Firebase Authentication aktivieren

1. Im linken Menü: **Build → Authentication → Jetzt loslegen**
2. Unter **Sign-in method**: **E-Mail/Passwort** aktivieren → Speichern

---

## Schritt 3 · Firestore Datenbank anlegen

1. Im linken Menü: **Build → Firestore Database → Datenbank erstellen**
2. Modus: **Produktionsmodus** (Regeln kommen im nächsten Schritt)
3. Standort: `europe-west3` (Frankfurt)

### Firestore Sicherheitsregeln

Unter **Firestore → Regeln** folgenden Code einfügen:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Nutzerdaten: jeder liest/schreibt nur eigene Daten
    // Lehrer & Schulleitung dürfen alle Schüler lesen und anlegen
    match /users/{uid} {
      allow read: if request.auth != null &&
        (request.auth.uid == uid ||
         get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ['lehrer','leitung']);
      allow write: if request.auth != null &&
        (request.auth.uid == uid ||
         get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ['lehrer','leitung']);
    }

    // Pläne: Schüler schreiben nur eigene; Lehrer & Leitung lesen alle
    match /plans/{planId} {
      allow write: if request.auth != null &&
        planId.matches(request.auth.uid + '_.*');
      allow read: if request.auth != null &&
        (planId.matches(request.auth.uid + '_.*') ||
         get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ['lehrer','leitung']);
    }
  }
}
```

---

## Schritt 4 · Web-App registrieren & Config holen

1. Firebase Konsole → Projektübersicht → **`</>`** (Web-App hinzufügen)
2. App-Nickname: `INA Web`
3. **„App registrieren"** → Du siehst ein Codeschnipsel wie:

```js
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "ina-gms-stwend.firebaseapp.com",
  projectId: "ina-gms-stwend",
  storageBucket: "ina-gms-stwend.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123...:web:abc..."
};
```

4. Diese Werte in `index.html` eintragen (Zeile mit `DEIN_API_KEY` usw.)

---

## Schritt 5 · Erste Konten anlegen (Schulleitung & Lehrer)

Da Browser-Apps keine Admin-Konten selbst erstellen können, legst du die
**Schulleitung und Lehrer-Konten einmalig manuell** an:

### 5a · Auth-Konto anlegen
Firebase Konsole → Authentication → Benutzer → **Benutzer hinzufügen**
- E-Mail: z. B. `schulleitung@gms-stwend.de`
- Passwort: beliebiges Startpasswort (wird beim ersten Login geändert)
- Notiere die angezeigte **UID**

### 5b · Firestore-Dokument anlegen
Firestore → Daten → Collection `users` → Dokument mit der UID als ID:

```json
{
  "vorname": "Maria",
  "nachname": "Meier",
  "email": "schulleitung@gms-stwend.de",
  "role": "leitung",
  "mustSetPassword": true
}
```

Für Lehrkräfte: `"role": "lehrer"`

---

## Schritt 6 · Schüler anlegen

Schüler werden von Lehrern oder der Schulleitung **direkt in der App** angelegt
(Tab „Schüler → Hinzufügen").

> **Wichtig:** Das Anlegen eines Firebase-Auth-Kontos für Schüler erfordert
> eine **Cloud Function** (weil der Browser keine fremden Auth-Konten anlegen kann).
> Bis die Cloud Function eingerichtet ist, Schüler-Auth-Konten ebenfalls
> manuell über die Firebase Konsole anlegen (wie in Schritt 5a beschrieben),
> Firestore-Dokument mit `"role": "schueler"` + `"klasse": "7a"` usw.

### Optional: Cloud Function für Schüler-Anlage

```js
// functions/index.js
const functions = require("firebase-functions");
const admin = require("firebase-admin");
admin.initializeApp();

exports.createStudent = functions.https.onCall(async (data, context) => {
  // Nur Lehrer & Leitung dürfen das
  const callerDoc = await admin.firestore().collection("users").doc(context.auth.uid).get();
  if (!["lehrer","leitung"].includes(callerDoc.data().role)) {
    throw new functions.https.HttpsError("permission-denied", "Keine Berechtigung.");
  }
  const user = await admin.auth().createUser({ email: data.email, password: data.pw });
  await admin.firestore().collection("users").doc(user.uid).set({
    vorname: data.vorname, nachname: data.nachname,
    klasse: data.klasse, email: data.email,
    role: "schueler", mustSetPassword: true,
    createdAt: new Date().toISOString()
  });
  return { uid: user.uid };
});
```

---

## Schritt 7 · Auf GitHub Pages veröffentlichen

1. GitHub-Repository anlegen (z. B. `ina-gms`)
2. `index.html` und `EINRICHTUNG.md` hochladen
3. Repository → **Settings → Pages → Branch: main / (root)** → Speichern
4. Nach 1–2 Minuten erreichbar unter:
   `https://DEIN-GITHUB-NAME.github.io/ina-gms/`

---

## Datenstruktur (Übersicht)

```
Firestore
├── users/
│   ├── {uid}          ← ein Dokument pro Person
│   │   ├── vorname, nachname, klasse, email
│   │   ├── role       ← "schueler" | "lehrer" | "leitung"
│   │   └── mustSetPassword  ← true beim ersten Login
│
└── plans/
    └── {uid}_{datum}  ← z. B. "abc123_2025-06-23"
        ├── uid, tag
        ├── zs1, zs2   ← frühe Angebote (oder null)
        └── zs3, zs4   ← späte Angebote (oder null)
```

---

## Fragen?

Bei technischen Fragen zur Einrichtung einfach Claude fragen –
alle Dateinamen und Strukturen sind genau wie hier beschrieben.

---

## Firestore-Regeln ergänzen (für Einstellungen)

Da die Schulleitung jetzt Schuljahres-Einstellungen speichert, muss die Firestore-Regel um die `config`-Collection erweitert werden. Füge diesen Block in deine bestehenden Regeln ein:

```
// Schuljahres-Einstellungen: nur Schulleitung schreibt, alle lesen
match /config/{docId} {
  allow read: if request.auth != null;
  allow write: if request.auth != null &&
    get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'leitung';
}
```

---

## Neue Firestore-Regeln (Fächer + Anwesenheit)

Ergänze diese Blöcke in deinen bestehenden Firestore-Regeln:

```
// Fächer-Verwaltung: nur Schulleitung schreibt, alle lesen
match /faecher/{fachId} {
  allow read: if request.auth != null;
  allow write: if request.auth != null &&
    get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'leitung';
}

// Anwesenheit: Lehrer schreiben, Schulleitung + Lehrer + betroffene Schüler lesen
match /anwesenheit/{docId} {
  allow write: if request.auth != null &&
    get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ['lehrer','leitung'];
  allow read: if request.auth != null;
}
```
