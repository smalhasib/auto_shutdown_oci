# Remediation Plan: auto_shutdown_oci → OCI Instance Principals

## Objetivo

Eliminar la API key RSA de OCI del archivo `.env` en stgo-1a migrando a **OCI Instance Principals**, donde la instancia se autentica por su propia identidad sin credenciales estáticas en disco.

**Hallazgo original (RED):** `/home/ubuntu/auto_shutdown_oci/.env` contiene `OCI_KEY_CONTENT` (RSA 2048-bit), `OCI_USER`, `OCI_TENANCY`, `OCI_FINGERPRINT`. Un atacante con acceso al nodo obtiene control total del tenancy.

---

## Fase 1: Local — Preparar archivos modificados

### 1.1. main.py modificado

**Archivo:** `auto_shutdown_oci_fix/main.py`

Cambiar `load_oci_config()` para que intente Instance Principal primero, con fallback a API key:

```python
def load_oci_config():
    """Priority: Instance Principal > env vars (API key) > ~/.oci/config file."""
    # Strategy 1: Instance Principal (no secrets on disk)
    try:
        signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
        region = get_metadata_value("canonicalRegionName") or get_metadata_value("region")
        if region:
            signer.region = region
        print("[INIT] Authenticated via OCI Instance Principal (no API key required).")
        return {"auth_type": "instance_principal", "signer": signer, "region": region}
    except Exception as e:
        print(f"[INFO] Instance Principal not available: {e}")

    # Strategy 2: Environment variables (API key - legacy fallback)
    try:
        required = ["OCI_USER", "OCI_TENANCY", "OCI_FINGERPRINT", "OCI_KEY_CONTENT"]
        if all(k in os.environ for k in required):
            key_content = os.environ["OCI_KEY_CONTENT"].replace("\\n", "\n")
            config = {
                "user": os.environ["OCI_USER"],
                "tenancy": os.environ["OCI_TENANCY"],
                "fingerprint": os.environ["OCI_FINGERPRINT"],
                "key_content": key_content,
                "region": os.environ.get("OCI_REGION", "sa-santiago-1"),
            }
            oci_config = oci.config.from_dict(config)
            print("[INIT] Authenticated via API key (legacy fallback).")
            return {"auth_type": "api_key", "config": oci_config, "region": oci_config["region"]}
    except Exception as e:
        print(f"[ERROR] Failed to load API key config: {e}")

    # Strategy 3
    try:
        oci_config = oci.config.from_file()
        print("[INIT] Authenticated via ~/.oci/config file.")
        return {"auth_type": "api_key", "config": oci_config, "region": oci_config["region"]}
    except Exception as e:
        print(f"[ERROR] Failed to load OCI config file: {e}")

    print("[ERROR] No OCI authentication method available.")
    return None
```

Y en `run_monitor()`, usar el config/signer según el auth_type:

```python
if auth_type == "instance_principal":
    signer = config["signer"]
    budget_client = oci.budget.BudgetClient(config={}, signer=signer)
    compute_client = oci.core.ComputeClient(config={}, signer=signer)
else:
    oci_config = config["config"]
    budget_client = oci.budget.BudgetClient(oci_config)
    compute_client = oci.core.ComputeClient(oci_config)
```

**Referencia:** GitHub original → https://github.com/danielretamal/auto_shutdown_oci

### 1.2. docker-compose.yml modificado

**Archivo:** `auto_shutdown_oci_fix/docker-compose.yml`

Marcar OCI_USER, OCI_TENANCY, OCI_FINGERPRINT, OCI_KEY_CONTENT como opcionales (legacy fallback). Agregar comentario para Instance Principal:

```yaml
version: '3.8'
services:
  auto-shutdown:
    build: .
    container_name: auto-shutdown
    restart: unless-stopped
    env_file:
      - .env
    # NOTA: Para OCI Instance Principal, el contenedor necesita acceder al
    # metadata endpoint de OCI (169.254.169.254). Si falla la conexión,
    # descomentar:
    # network_mode: "host"
```

---

## Fase 2: OCI IAM — Dynamic Group + Policy

### 2.1. Dynamic Group

Crear un Dynamic Group que matchee la instancia stgo-1a:

```bash
oci iam dynamic-group create \
  --name "auto-shutdown-dg" \
  --matching-rule "instance.id = 'ocid1.instance.oc1.sa-santiago-1.anzwgljrpqq7h6acrjnudid2odx47opcvctzdkfgfbbmcyozwtdkyib7g2na'" \
  --description "auto_shutdown_OCI instances (stgo-1a)"
```

**Para expandir a otros nodos**, usar OR:
```
instance.id = 'ocid1....stgo-1a' || instance.id = 'ocid1....stgo-1b' || instance.id = 'ocid1....valpo-1'
```

### 2.2. IAM Policy

Crear política de mínimo privilegio:

```bash
oci iam policy create \
  --name "auto-shutdown-policy" \
  --statements '[
    "Allow dynamic-group auto-shutdown-dg to read budgets in tenancy",
    "Allow dynamic-group auto-shutdown-dg to manage instances in tenancy where request.instanceId = target.instance.id"
  ]' \
  --description "Minimal permissions for auto_shutdown_OCI"
```

### 2.3. Verificar IAM

```bash
oci iam dynamic-group list --all --query "data[?name=='auto-shutdown-dg']"
oci iam policy list --all --query "data[?name=='auto-shutdown-policy']"
```

---

## Fase 3: Deploy a stgo-1a

### 3.1. SCP los archivos modificados

```bash
scp -i credenciales/stgo-1.key auto_shutdown_oci_fix/main.py ubuntu@146.181.54.57:/home/ubuntu/auto_shutdown_oci/main.py
scp -i credenciales/stgo-1.key auto_shutdown_oci_fix/docker-compose.yml ubuntu@146.181.54.57:/home/ubuntu/auto_shutdown_oci/docker-compose.yml
```

### 3.2. Sanitizar .env (remoto vía SSH)

Eliminar las 4 variables de API key OCI, conservar el resto:

```bash
ssh -i credenciales/stgo-1.key ubuntu@146.181.54.57
cd /home/ubuntu/auto_shutdown_oci
# Backup por si algo sale mal
cp .env .env.bak.$(date +%Y%m%d_%H%M%S)
# Eliminar las 4 líneas de API key OCI
sed -i '/^OCI_USER=/d' .env
sed -i '/^OCI_TENANCY=/d' .env
sed -i '/^OCI_FINGERPRINT=/d' .env
sed -i '/^OCI_KEY_CONTENT=/d' .env
```

### 3.3. Rebuild y restart del contenedor

```bash
cd /home/ubuntu/auto_shutdown_oci
sudo docker compose down
sudo docker compose build --no-cache
sudo docker compose up -d
```

---

## Fase 4: Verificación

### 4.1. Logs del contenedor

```bash
ssh -i credenciales/stgo-1.key ubuntu@146.181.54.57 "sudo docker compose logs -f --tail 100"
```

Buscar en los logs:
- `[INIT] Authenticated via OCI Instance Principal (no API key required).` → Éxito
- O `[INFO] Instance Principal not available: ...` → Falló, ver policy/network
- Si falla, el fallback a API key sigue funcionando

### 4.2. Forzar chequeo inmediato

Si el script expone un endpoint o señal, forzar el chequeo. Si no, esperar CHECK_INTERVAL (1800s = 30 min). Dado que `SIMULATE_OVER_BUDGET=true` y `DRY_RUN=true`, no hay riesgo de acción real.

### 4.3. Confirmar que el .env ya no tiene API key

```bash
ssh -i credenciales/stgo-1.key ubuntu@146.181.54.57 "grep -E '^OCI_(USER|TENANCY|FINGERPRINT|KEY_CONTENT)=' /home/ubuntu/auto_shutdown_oci/.env || echo 'OK: No OCI API keys in .env'"
```

### 4.4. Telegram

Confirmar que el bot envía notificaciones normalmente. El chat_id y bot_token se conservan.

---

## Fase 5: Cleanup (opcional)

- Eliminar el backup del .env después de confirmar que todo funciona:
  ```bash
  rm /home/ubuntu/auto_shutdown_oci/.env.bak.*
  ```
- Rotar la API key en OCI Console (ya no se usará)
- Una vez rotada, eliminar cualquier copia local de la key

---

## Rollback Plan

Si algo falla:

```bash
ssh -i credenciales/stgo-1.key ubuntu@146.181.54.57
cd /home/ubuntu/auto_shutdown_oci
# Restaurar backup
cp .env.bak.* .env
# Revertir main.py desde GitHub
curl -o main.py https://raw.githubusercontent.com/danielretamal/auto_shutdown_oci/main/main.py
sudo docker compose down && sudo docker compose build --no-cache && sudo docker compose up -d
```

---

## Estado Actual

| Paso | Estado |
|------|--------|
| 1.1 main.py modificado | ✅ Completado |
| 1.2 docker-compose.yml modificado | ✅ Completado |
| 2.1 Dynamic Group | ✅ Completado |
| 2.2 IAM Policy | ✅ Completado |
| 3.1 SCP archivos | ✅ Completado |
| 3.2 Sanitizar .env | ✅ Completado |
| 3.3 Rebuild Docker | ✅ Completado |
| 4.1 Verificar logs | ✅ Completado |
| 4.2 Forzar chequeo | ✅ Completado |
| 4.3 Confirmar .env limpio | ✅ Completado |
| 5 Cleanup | ✅ Completado |
