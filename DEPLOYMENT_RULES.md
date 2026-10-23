# Wymagania Wdrożeniowe Platformy (Deployment Rules)

> ⚠️ **OSTRZEŻENIE:** Wszystkie manifesty wdrażane na platformę przechodzą automatyczną weryfikację przed uruchomieniem. Złożenie manifestów niespełniających poniższych reguł spowoduje **natychmiastowe zablokowanie wdrożenia (`SyncFailed` / `Admission Denied`)**.

---

## 1. Wymagany kontekst bezpieczeństwa kontenerów (Non-Root)

Każdy `Deployment`, `StatefulSet`, `Job` oraz `CronJob` **musi** uruchamiać kontenery jako użytkownik nieuprzywilejowany (`non-root`) z wyłączoną eskalacją uprawnień i zrzuconymi capabilities systemu Linux:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          securityContext:
            privileged: false
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            capabilities:
              drop: ["ALL"]
            seccompProfile:
              type: RuntimeDefault
```

**Co powoduje blokadę wdrożenia:**
- Brak `runAsNonRoot: true` lub próba uruchomienia kontenera jako `root` (`UID 0`).
- Brak `allowPrivilegeEscalation: false` lub `capabilities.drop: ["ALL"]`.
- Użycie `privileged: true`, `hostNetwork: true`, `hostPID: true`, `hostIPC: true` lub wolumenów `hostPath`.

---

## 2. Limity zasobów i Kwoty Środowiska (Resource Quotas)

Każdy kontener powinien definiować `resources.requests` oraz `resources.limits` mieszczące się w przydzielonej kwocie środowiska (np. `4 vCPU`, `4 GiB RAM`, `40 GiB Storage`):

```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

**Co powoduje blokadę wdrożenia:**
- Przekroczenie sumarycznego limitu vCPU, pamięci RAM lub przestrzeni dyskowej (`ceph-rbd`) przyznanego dla danego namespace'u.

---

## 3. Dozwolone i zabronione zasoby Kubernetes

Wdrożenia działają wyłącznie w obrębie przypisanego namespace'u (np. `mtechnology-demo`).

- **Dozwolone zasoby:** `Deployment`, `StatefulSet`, `Job`, `CronJob`, `Service`, `ConfigMap`, `Secret`, `ExternalSecret`, `PersistentVolumeClaim` (`storageClassName: ceph-rbd`), `HorizontalPodAutoscaler`, `PodDisruptionBudget`, `ServiceAccount`, `HTTPRoute`, `Ingress`.
- **Zasoby zabronione (blokowane przy wdrożeniu):**
  - Tworzenie lub modyfikacja `Namespace`, `ResourceQuota`, `LimitRange`, `NetworkPolicy`, `SecretStore` oraz jakichkolwiek zasobów klaster-scoped (`ClusterRole`, `ClusterRoleBinding`, `CustomResourceDefinition`, `PersistentVolume`).
  - Modyfikacja zasobów systemowych zarządzanych przez platformę.
  - Tworzenie ról RBAC (`Role` / `RoleBinding`) z uprawnieniami do zapisu lub odczytu `secrets` (dozwolone są wyłącznie role typu *read-only peer discovery*: `get`, `list`, `watch` dla `pods`, `services`, `endpoints`, `endpointslices`, `configmaps`).

---

## 4. Sekrety (Vault) i Pamięć Trwała (Storage)

- **Sekrety z Vault:** Do pobierania sekretów z platformowego Vaulta należy używać zasobu `ExternalSecret` ze wskazaniem na `secretStoreRef.name: tenant-vault-store` oraz ścieżką w obrębie przypisanego środowiska (`org-<prefix>/kv/data/namespaces/<sufiks>/app/<klucz>`). Odwołanie do ścieżki innego środowiska lub innej organizacji zostanie odrzucone.
- **Wolumeny trwałe (PVC):** Dla `PersistentVolumeClaim` należy używać klasy `storageClassName: ceph-rbd`.
