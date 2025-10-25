# Week 10, Day 3, Task 2081: Install FCM Package

**Tydzień**: 10  
**Dzień**: 3 (Środa)  
**Zadanie**: 2081  
**Typ**: Package Installation  
**Status**: ✅ Do wykonania

---

## 🎯 Cel

Dodanie Firebase Admin SDK dla server-side push notifications

---

## 📋 Kontekst

FirebaseAdmin.Messaging pozwala na sending push notifications do iOS i Android devices z backend. FCM jest free, reliable service od Google z support dla rich notifications (images, actions, custom data). 

Alternative to expensive services jak OneSignal czy Pusher. FCM handles token management i delivery automatically, retry logic dla failed deliveries, oraz analytics o notification delivery rates.

---

## 🔧 Rola w Systemie

**Core notification infrastructure** - enables push to mobile devices.

Ten package jest foundation dla całego push notification system. Pozwala na:
- Sending notifications do pojedynczych devices
- Broadcasting do topics (grupy użytkowników)
- Scheduling notifications
- Tracking delivery status

---

## 💻 Polecenie

```bash
cd src/Abelado.Infrastructure
dotnet add package FirebaseAdmin
dotnet add package FirebaseAdmin.Messaging
```

---

## ✅ Weryfikacja

Sprawdź że packages zostały dodane do `Abelado.Infrastructure.csproj`:

```xml
<PackageReference Include="FirebaseAdmin" Version="2.4.0" />
<PackageReference Include="FirebaseAdmin.Messaging" Version="2.4.0" />
```

---

## 📊 Zależności

**Wymaga**:
- .NET 8.0
- Google.Apis.Auth (dependency FirebaseAdmin)

**Używane przez**:
- Zadanie 2082 (Firebase configuration)
- Zadanie 2083 (Push notification service)

---

## 🔗 Powiązane Zadania

- **Poprzednie**: Task 2080 (Advanced search testing)
- **Następne**: Task 2082 (Configure Firebase Admin SDK)
- **Zależne**: Task 2083, 2084, 2085 (notification services)

---

## 📝 Notatki

- Firebase Admin SDK jest **server-side only** - nie używać w mobile apps
- Wersja 2.4.0+ wspiera .NET 8.0
- Package size: ~2MB
- Free tier FCM: unlimited notifications
- Alternative packages: OneSignal.NetStandard (paid), Pusher (paid)
