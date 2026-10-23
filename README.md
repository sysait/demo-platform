# Demo Platform — Statystyki Przeglądarek (NATS + UI)

Prosty przykład wdrożenia składający się z 3 lekkich komponentów komunikujących się przez kolejkę **NATS**:

1. **`nats`** (`nats.yaml`) — broker wiadomości NATS (`nats:4222`).
2. **`browser-publisher`** (`publisher.yaml`) — usługa publikująca w pętli zdarzenia wizyt przeglądarek (`Chrome`, `Edge`, `Safari`, `Firefox`, `Other`) na temat `browsers.visits`.
3. **`browser-analytics-ui`** (`consumer-ui.yaml`) — usługa subskrybująca temat `browsers.visits` z NATS i wyświetlająca na żywo statystyki przeglądarek pod adresem `https://demo.sysait.com` (z możliwością ręcznego wysłania zdarzenia `+1` z poziomu UI).

> ⚠️ Przed wdrożeniem własnych usług zapoznaj się z dokumentem **[DEPLOYMENT_RULES.md](./DEPLOYMENT_RULES.md)** opisującym wymagania bezpieczeństwa i zasobów. Manifesty niespełniające tych zasad są automatycznie blokowane przez platformę.

---

## Konfiguracja zmiennych (`vars.properties`)

Docelowy namespace, prefiks organizacji oraz domena ustawiane są w jednym pliku [`vars.properties`](./vars.properties):

```ini
NAMESPACE=mtechnology-demo
ORG_PREFIX=mtechnology
ENV_SUFFIX=demo
INGRESS_HOST=demo.sysait.com
GATEWAY_NAME=cf-lcj11-a-public
```

Podgląd wygenerowanych manifestów:

```bash
kubectl kustomize .
```
