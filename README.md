# SysAIT Demo Platform — Cyber Snake 🐍

Lekka aplikacja demonstracyjna w architekturze **GitOps** przygotowana dla platformy **SysAIT**. Aplikacja prezentuje pełen cykl życia wdrożenia mikrousługi na platformie: od utworzenia izolowanego środowiska, przez bezpieczne wstrzyknięcie hasła ze skarbca **Platform Secret Vault** (`OpenBao`) za pomocą `stctl`, aż po deklaratywną synchronizację GitOps i wystawienie publicznego punktu wejściowego (Ingress / Gateway API).

---

## 🎮 Czym jest Cyber Snake?

Aplikacja to retro-cyberpunkowa gra w wężyka (HTML5 Canvas + Web API) z wbudowanym mechanizmem kontroli dostępu (**Terminal Access Lock**):
- **Weryfikacja hasła:** Aby uruchomić grę, użytkownik musi wprowadzić hasło dostępu.
- **Hasło ze skarbca Vault:** Wartość hasła (`GAME_PASSWORD`) jest wstrzykiwana bezpośrednio do poda z szyfrowanego skarbca **Platform Secret Vault** za pośrednictwem `stctl secrets put` i `stctl secrets bind`.
- **Domyślne hasło demonstracyjne:**
  ```text
  SysAIT-Snake-2026
  ```
- **Automatyczny restart po rotacji (Reloader):** Dzięki adnotacji `reloader.stakater.com/auto: "true"`, każda zmiana hasła w `stctl` powoduje natychmiastowy, bezprzerwowy restart poda i zaktualizowanie klucza w locie!

---

## 🚀 Procedura Wdrożenia Zero-to-One (`stctl`)

Poniższa procedura odzwierciedla oficjalny standard wdrożeniowy platformy SysAIT (`sysait://guides/application_bootstrap`):

### 1. Sprawdzenie kwot i tożsamości
Upewnij się, że jesteś zalogowany do CLI `stctl` oraz że organizacja posiada wolne zasoby:
```bash
stctl whoami
stctl quota get
```

### 2. Utworzenie izolowanego środowiska (Namespace)
Utwórz dedykowany namespace dla środowiska demonstracyjnego (`demo`):
```bash
stctl ns create demo \
  --type development \
  --cpu 1 \
  --mem 2 \
  --storage 10
```
> Platforma automatycznie utworzy zabezpieczony namespace Kubernetes (np. `mdm-demo`).

### 3. Wstrzyknięcie hasła gry do Platform Secret Vault
Narzędzie `stctl` realizuje politykę **Zero Secret Intermediation** (tajemnice i hasła nigdy nie są odczytywane przez agentów ani logowane w postaci jawnej):
```bash
stctl secrets put \
  --env demo \
  --key game \
  --kv GAME_PASSWORD=SysAIT-Snake-2026
```

### 4. Związanie sekretu ze środowiskiem (SecretBinding)
Utwórz wiązanie, które bezpiecznie zmapuje klucz z Vaulta do natywnego sekretu Kubernetes `snake-game-secret` w namespace środowiska:
```bash
stctl secrets bind \
  --name snake-game-binding \
  --env demo \
  --vault-path game \
  --target-secret snake-game-secret \
  --key GAME_PASSWORD
```

### 5. Rejestracja i synchronizacja aplikacji GitOps
Zarejestruj repozytorium w kontrolerze GitOps platformy z automatyczną synchronizacją i samonaprawianiem:
```bash
stctl apps create snake-game \
  --env demo \
  --repo https://github.com/sysait/demo-platform.git \
  --revision main \
  --path . \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

Wymuś natychmiastową synchronizację i zweryfikuj status zdrowia aplikacji:
```bash
stctl apps sync snake-game
stctl apps get snake-game
```

### 6. Wystawienie punktu wejściowego (Endpoint / Ingress)
Zgodnie z polityką bezpieczeństwa dla środowisk typu `development`, dostęp początkowo zawężamy do adresu IP administratora (`curl -4 ifconfig.me`):
```bash
USER_IP="$(curl -4 -s https://ifconfig.me)/32"

stctl domains expose \
  --env demo \
  --name snake \
  --service snake-game \
  --port 80 \
  --protocol HTTP \
  --allow-ips "${USER_IP}"
```
Po wykonaniu polecenia otrzymasz unikalny adres URL (np. `https://snake-<envId>.stctl.com`), pod którym gra jest dostępna.

---

## 🔐 Zarządzanie Hasłem w Czasie Rzeczywistym

Chcesz zmienić hasło odblokowujące grę? Wystarczy zaktualizować sekret w `stctl`:
```bash
stctl secrets put --env demo --key game --kv GAME_PASSWORD=MojeNoweSuperHaslo2026
```
Wiązana tajemnica zostanie automatycznie zaktualizowana w klastrze, a pod zrestartowany bez utraty dostępności.

---

## 🛡️ Bezpieczeństwo i Reguły Wdrożeniowe (Guardrails)

Wszystkie manifesty w tym repozytorium są zgodne z wymogami **[DEPLOYMENT_RULES.md](./DEPLOYMENT_RULES.md)**:
- **Kontekst kontenera:** `runAsNonRoot: true`, `runAsUser: 1000`, `allowPrivilegeEscalation: false`, `capabilities: drop ["ALL"]`, `seccompProfile: RuntimeDefault`.
- **Zasoby:** Zdefiniowane `requests` (50m CPU, 64Mi RAM) oraz `limits` (200m CPU, 128Mi RAM).
- **Proby:** Zdefiniowane `readinessProbe` i `livenessProbe` na ścieżce `/health` (port 8080).
