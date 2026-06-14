# SEMANA 10 - Monitoreo y Observabilidad

**Estudiante:** Gianfranco Campos Acevedo - **ID:** 000274878

## INSTRUCCIONES
## 1. Clonar el repositorio y Preparar el Entorno (Docker Nativo)

Clonamos el repositorio dado por el docente.

Debido a una incompatibilidad de cAdvisor corriendo bajo Docker Desktop en Windows (el cual aísla físicamente los *cgroups* impidiendo la recolección de métricas y la inyección de la etiqueta `name`), se tomó la decisión técnica de desplegar un entorno de Docker nativo directamente sobre el subsistema de Linux (WSL2 con Ubuntu 24.04).

Para preparar el entorno nativo en Ubuntu, ejecutamos la instalación oficial de Docker Engine, bloqueamos su versión para evitar conflictos y configuramos el driver de almacenamiento a `overlay2`:

```bash
# Instalación de dependencias y llaves GPG de Docker
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Agregar repositorio e instalar Docker CE nativo
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce=5:26.1.4-1~ubuntu.24.04~noble docker-ce-cli=5:26.1.4-1~ubuntu.24.04~noble containerd.io docker-buildx-plugin docker-compose-plugin

# Configuración del driver y permisos
cat <<'EOF' | sudo tee /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
EOF
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl restart docker

```

## 2. Levantar los contenedores 

```bash
sudo mount --make-rshared /
docker compose up -d --build
```

Tras esto, validamos que todos los contenedores estén operativos:

```bash
docker ps
```

## 3. Acceso a las aplicaciones

A través de nuestro navegador accedemos mediante `localhost` a los puertos definidos en el `docker-compose.yml`:

* **Frontend:** `localhost:8080`
* **Backend (API):** `localhost:3001`
* **Grafana:** `localhost:3000`
* **Prometheus:** `localhost:9090`
* **cAdvisor:** `localhost:8081`

## 4. Generar métricas iniciales

Para iniciar con la recolección, nos dirigimos al Frontend y presionamos el botón "Saludar API" varias veces para generar peticiones. Para visualizar la data cruda recolectada, ingresamos al endpoint exigido por el protocolo de scraping de Prometheus:
`localhost:3001/metrics`

## 5. Creación de Dashboards

En Grafana, creamos un nuevo Dashboard con los siguientes 4 paneles fundamentales:

### Panel 1: CPU del contenedor de la aplicación

* **Fuente:** Prometheus | **Visualización:** Time series | **Unidad:** Percent (0-100)
* **PromQL:**
```promql
sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
```

### Panel 2: CPU del host (Infraestructura)

* **Fuente:** Prometheus | **Visualización:** Time series | **Unidad:** Percent (0-100)
* **PromQL:**

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

### Panel 3: Logs de aplicación (API + Frontend)

* **Fuente:** Loki | **Visualización:** Logs
* **LogQL:**

```logql
{tier="application"} | json
# Ejemplo para filtrar solo errores: {tier="application"} | json | level="ERROR"
```

### Panel 4: Logs de infraestructura

* **Fuente:** Loki | **Visualización:** Logs
* **LogQL:**

```logql
{tier="infrastructure"}
```

## 6. Configuración de alerta

Se procedió a configurar una regla en Grafana Alerting para monitorear el consumo de CPU del backend:

* **Query:** `sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100`
* **Condición (Threshold):** `IS ABOVE 50`
* **Pending period:** `30s`
* **Etiquetas:** `severity = warning`

Al generar carga desde el frontend, la gráfica superará el 50% y la alerta pasará por los estados *Normal* -> *Pending* -> *Firing*.

## 7. Cerrar el ciclo de la alarma (Webhook -> Log)

Para evidenciar el ciclo completo de observabilidad, configuramos un *Contact Point* en Grafana de tipo **Webhook** apuntando a la ruta interna de nuestro backend:

```text
http://backend:3001/alerts
```

Se actualizó la *Notification Policy* para enviar los disparos a este nuevo Webhook. Para demostrar de manera irrefutable que el backend recibió la alerta y generó un log transaccional visible en Grafana, utilizamos el siguiente filtro en nuestro panel de **Logs de Aplicación**:

```logql
{tier="application"} | json | msg="grafana_alert_received"
```

**Trazabilidad en la infraestructura:**
Si deseamos ver los registros internos de Grafana confirmando el envío exitoso de la alerta, filtramos en el panel de **Logs de Infraestructura**:

```logql
{tier="infrastructure"} |= "alert"
```

Se valida el flujo total: **Métrica cruza umbral -> Grafana genera Alarma -> Disparo de Webhook -> Backend registra Log -> Loki lo visualiza en el Dashboard.**


## PREGUNTAS

### 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?
Prometheus recolecta métricas numéricas que revelan cuándo ocurre un problema y cuál es su magnitud (el síntoma), pero no explica el motivo. Loki complementa esto indexando los logs y stack traces de la aplicación, aportando el contexto textual necesario para entender el porqué (la causa raíz). En conjunto, Prometheus dispara la alerta de la anomalía y Loki proporciona la evidencia para diagnosticarla y resolverla.

### 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?
Aplicar Infraestructura como Código (IaC) automatiza el despliegue, haciéndolo repetible y eliminando errores humanos. Al definir las conexiones en la configuración, Grafana arranca con las fuentes de datos operativas desde el primer segundo, sin requerir intervención manual en su interfaz. Esto facilita la clonación de laboratorios, la migración rápida entre entornos (Dev/Prod) y permite auditar cualquier cambio a través del control de versiones en Git.

### 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?
Difieren porque monitorean distintos niveles de aislamiento. La CPU del host mide el consumo global de la máquina entera (incluyendo todos los contenedores y procesos del sistema operativo), mientras que la del contenedor aísla exclusivamente los recursos que consume ese servicio en particular.
Para alertar sobre una aplicación usaría la CPU del contenedor, ya que refleja su estado real y evita falsos positivos en caso de que otro proceso externo sature la máquina.

### 4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?
La diferencia está en la etapa de evaluación:
* El evaluation interval es la frecuencia con la que Grafana ejecuta la consulta para verificar si se cumple la regla (por ejemplo, evaluar la métrica cada 10 segundos).
* El pending period es el tiempo de espera continuo que la métrica debe mantenerse violando el umbral antes de activar definitivamente la alarma (Firing). Su propósito es filtrar falsos positivos causados por picos de consumo momentáneos o micro-saturaciones.