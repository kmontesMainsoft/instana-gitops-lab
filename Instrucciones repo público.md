# Laboratorio: Instana Host Agent + Git-based Configuration Management

**Objetivo:** tener un host de prueba cuyo `configuration.yaml` viva en un repo de GitHub, poder cambiar un tag ahí, y ver cómo se aplica al host.

---

## Prerrequisitos

- Un host Linux (VM, contenedor, o tu laptop) con el **Instana host agent ya instalado y corriendo**.
- Acceso a la consola de Instana (sandbox o tu tenant).
- Una cuenta de GitHub y `git` instalado en tu máquina.
- Un **API Token** de Instana con permiso "Configuration of agents" (Settings → API Tokens).

---

## Paso 1 — Crear el repositorio en GitHub

1. Crea un repo nuevo, por ejemplo `instana-gitops-lab`.
2. Clónalo en tu laptop:

   ```bash
   git clone https://github.com/<tu-usuario>/instana-gitops-lab.git
   cd instana-gitops-lab
   ```

---

## Paso 2 — Crear la estructura de carpetas correcta

**Regla de oro:** el repo debe reflejar la misma estructura que existe dentro de `<instanaAgentDir>/etc/` en el host. Casi siempre eso significa una carpeta `instana/`.

```bash
mkdir instana
```

Crea `instana/configuration.yaml` con algo simple para probar, por ejemplo:

```yaml
com.instana.plugin.generic.hardware:
  enabled: true

com.instana.agent.main.config.Host:
  tags:
    - lab
    - version-1
```

(Usa contenido real de tu `configuration.yaml` actual si ya tienes uno — cópialo tal cual a `instana/configuration.yaml`.)

Tu repo debe verse así:

```
instana-gitops-lab/
└── instana/
    └── configuration.yaml
```

---

## Paso 3 — Subir el repo a GitHub

```bash
git add .
git commit -m "config inicial del lab"
git push origin main
```

Con esto, tu repo en GitHub ya tiene el archivo. Este es el paso que se repite cada vez que quieras cambiar algo (edites en tu laptop o directo en la web de GitHub, es lo mismo: ambos terminan en un commit en `main`).

---

## Paso 4 — Conectar el host a ese repo (la parte "Initialize")

Esta es la pantalla que viste en Instana:

> Configuration management → Initialize → "This agent does not use git-based configuration management"

Pasos:

1. En Instana, ve a **Infrastructure → tu host → Agent Dashboard**.
2. Busca la sección **Configuration Management** y da clic en **Initialize**.
3. Completa:
   - **Repository URL:** `https://github.com/<tu-usuario>/instana-gitops-lab.git`
   - **Branch:** `main`
   - Credenciales si el repo es privado (usuario + token de acceso personal de GitHub).
4. Clic en **Initialize & Restart**.

El agente se reinicia, clona tu repo dentro de su carpeta interna, y queda "casado" con ese repo + rama.

> Alternativa por API (útil si luego quieres automatizar con muchos hosts):
> 
> ```bash
> curl --request POST \
>   --url "https://<tu-tenant>.instana.io/api/host-agent/configuration?query=entity.host.name:LABS" \
>   --header "authorization: apiToken $INSTANA_API_TOKEN" \
>   --header "content-type: application/json" \
>   --data '{
>     "remoteUri": "https://github.com/<tu-usuario>/instana-gitops-lab.git",
>     "remoteBranch": "main",
>     "remoteName": "configuration"
>   }'
> ```
> 
> Esto hace lo mismo que el botón, pero de forma programática.

---

## Paso 5 — Verificar que se conectó bien

En el host, entra a la carpeta interna del agente (ajusta la ruta según tu instalación, típicamente `/opt/instana/agent`):

```bash
cd /opt/instana/agent/etc
git remote -v
```

Deberías ver un remoto llamado `configuration` apuntando a tu repo de GitHub. Y dentro de `etc/instana/configuration.yaml` debería estar el contenido que subiste.

Revisa también el log del agente para confirmar que no hubo errores de autenticación:

```bash
tail -f /opt/instana/agent/data/log/agent.log
```

---

## Paso 6 — Hacer un cambio y aplicarlo (la prueba real)

1. Edita `instana/configuration.yaml` en tu repo — por ejemplo cambia `version-1` por `version-2` en los tags.
2. Push:

   ```bash
   git add .
   git commit -m "cambio de tag de prueba"
   git push origin main
   ```
3. El host **no se entera solo todavía**. Dispara la actualización de una de estas formas:
   - Reiniciando el servicio en el host:

     ```bash
     sudo systemctl restart instana-agent
     ```
   - O llamando a la API **sin** los datos del repo (ya está conectado, solo necesitas decirle a quién avisar):

     ```bash
     curl --request POST \
       --url "https://<tu-tenant>.instana.io/api/host-agent/configuration?query=entity.host.name:LABS" \
       --header "authorization: apiToken $INSTANA_API_TOKEN"
     ```
4. Verifica en la consola de Instana (pestaña Tags del host) que el tag cambió.

---

## Paso 7 (opcional/avanzado) — Automatizarlo con GitHub Actions

Una vez que el Paso 4 y 6 te funcionan manualmente, puedes crear `.github/workflows/update-agent.yml` en tu repo para que, en cada push a `main`, dispare automáticamente el paso 6.3 (la llamada **sin** `remoteUri`, solo con el `query`). Así el flujo real queda: cambias el tag → push → en segundos se aplica solo, sin que tengas que correr el curl a mano.

Guarda tu `INSTANA_API_TOKEN` como secret del repo (`Settings → Secrets and variables → Actions`) antes de hacer esto.

---

## Errores comunes (troubleshooting)

| Síntoma | Causa probable |
| --- | --- |
| El agente no clona el repo | Credenciales mal puestas, o repo privado sin token |
| El archivo aparece pero no se aplica | Está en la ruta equivocada dentro del repo (no coincide con la estructura de `etc/`) |
| El tag no cambia tras el push | Falta el paso de "avisar" al agente (push por sí solo no reinicia nada) |
| Todo funciona en un host pero no en otro | Cada host se conecta individualmente (Paso 4); revisa que también hiciste el Initialize en ese host |

---

## Resumen mental

- **Repo de GitHub** = fuente de la verdad. Vive ahí tu `configuration.yaml`.
- **Push** = solo actualiza el repo, no toca ningún host todavía.
- **"Avisar" al agente** (reinicio o API) = paso obligatorio para que el cambio se aplique.
- **GitHub Action** = automatiza ese "aviso" para que se sienta instantáneo.