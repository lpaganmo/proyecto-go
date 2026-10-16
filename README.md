 Portafolio IT en Go (UOC) — listo para Docker y Kubernetes

Portafolio de técnico IT con secciones **About me, Skills, Experience y Contact**.
Servidor en **Go (solo librería estándar, 0 dependencias)**. Paleta: azul UOC + colores de Google.

## Estructura y por qué existe cada archivo

| Archivo | Para qué sirve |
|---|---|
| `main.go` | Arranque, configuración por variables de entorno, timeouts y apagado ordenado (SIGTERM). |
| `server.go` | Rutas, handlers, probes `/healthz` `/readyz`, middlewares de logging, recover y cabeceras de seguridad. |
| `contact.go` | Validación d#el formulario, honeypot anti-bots y rate limiting por IP. |
| `content.go` | **Tus datos** (nombre, skills, experiencia). Es lo primero que debes editar. |
| `web/templates/index.html` | Plantilla HTML (html/template, escapado automático anti-XSS). |
| `web/static/` | CSS, JS y favicon, incrustados en el binario con `go:embed`. |
| `server_test.go` | Pruebas automáticas (para CI antes de construir la imagen). |
| `Dockerfile` / `.dockerignore` | Imagen multi-stage, distroless, usuario no-root. |
| `k8s/` | ConfigMap, Deployment, Service, Ingress y HPA. |

Cada bloque de código incluye un comentario con el **porqué** y el **para qué**.

## 1. Ejecutar en local

```bash
go test ./...
go run .
# http://localhost:8080
```

## 2. Docker

```bash
docker build -t portfolio-it:1.0.0 --build-arg VERSION=1.0.0 .
docker run --rm -p 8080:8080 portfolio-it:1.0.0
```

## 3. Kubernetes (ejemplo con Minikube)

```bash
minikube start
minikube addons enable ingress
minikube addons enable metrics-server
minikube image load portfolio-it:1.0.0      # evita usar un registro externo
# En k8s/10-deployment.yaml cambia image: por  portfolio-it:1.0.0
kubectl apply -f k8s/
kubectl get pods,svc,ingress
# Añade "127.0.0.1 portfolio.local" al fichero hosts y ejecuta: minikube tunnel
```

## Variables de entorno

| Variable | Por defecto | Descripción |
|---|---|---|
| `PORT` | `8080` | Puerto de escucha. |
| `TRUST_PROXY` | `false` | `true` detrás de Ingress para leer la IP real (`X-Forwarded-For`). |
| `CONTACT_RATE_LIMIT` | `5` | Envíos por IP cada 10 min (por réplica). |

## Decisiones pensadas para contenedores

- **Binario estático + `go:embed`**: imagen final de pocos MB, sin shell.
- **Sin estado**: no escribe en disco (`readOnlyRootFilesystem`); los mensajes del formulario salen por stdout en JSON.
- **Probes** separadas de liveness y readiness; **graceful shutdown** compatible con rolling updates.
- **Seguridad**: no-root, `drop ALL`, CSP estricta, timeouts, límite de cuerpo, honeypot y rate limit.

## CI/CD (GitHub Actions)

`.github/workflows/ci.yml`: **formato + vet + tests (-race) → build de imagen → push a GHCR**.
- En *pull requests* solo prueba y construye; en `main` y tags `v*` también publica `ghcr.io/<usuario>/<repo>`.
- Usa el `GITHUB_TOKEN` automático: no hace falta crear secretos.
- Tras el primer push, pon el paquete en GHCR como público (o crea un `imagePullSecret`) y ajusta `image:` en `k8s/10-deployment.yaml` a esa ruta (en minúsculas) y a una etiqueta concreta, p. ej. `sha-abc1234` o `1.0.0`.
- El despliegue (`kubectl apply`) queda manual a propósito; automatizarlo (CD) requiere credenciales del clúster.

## Siguientes pasos (ideas de ampliación)

1. Entrega real del contacto (SMTP o webhook) guardando credenciales en un `Secret` de Kubernetes.
2. CD automático hacia el clúster (p. ej. Argo CD o un job con `kubectl`).
3. HTTPS con cert-manager y métricas Prometheus (`/metrics`).
4. Sustituir los textos de `content.go` por tus datos reales y revisar el azul UOC con la guía de marca.
