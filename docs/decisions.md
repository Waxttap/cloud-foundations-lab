# Decision log

## Formato

```text
Decision:
Contexto:
Alternativas:
Tradeoff:
Resultado:
```

## Decisiones

### 001 - Laboratorios locales

Decision: usar Docker Compose, MinIO y LocalStack en lugar de cuentas AWS personales.

Contexto: evitar costos accidentales y reducir friccion de setup.

Tradeoff: no se practica consola AWS real en profundidad.

Resultado: los labs son reproducibles y reutilizables.

### 003 - Formato de eventos crudos

Decision: JSONL (JSON Lines) para data/raw/events.jsonl.

Contexto: los eventos se generan uno por vez. JSONL permite procesar con streaming
sin cargar todo el archivo en memoria, y es fácil de appender.

Alternativas: JSON array, CSV, Parquet.

Tradeoff: JSONL no es legible de un vistazo como un JSON array formateado.
Parquet sería más eficiente a escala, pero requiere dependencias externas.

Resultado: JSONL para raw. CSV para processed (compatibilidad analítica máxima).

### 004 - Pipeline de procesamiento

Decision: script Python (process_events.py) lee JSONL y escribe JSON filtrado.

Contexto: necesitamos filtrar un subconjunto de eventos GitHub Archive para análisis.
El script es reproducible: misma entrada, misma salida, sin efectos secundarios.

Tradeoff: un script por transformación vs una sola función general.
Elegimos un script por transformación: más legible, más fácil de testear.

Resultado: process_events.py → data/processed/push_events.json (filtra PushEvent)

### 002 - Entorno de desarrollo

Decision: GitHub Codespaces.

Contexto: el grupo no tiene instalaciones homogéneas (mix de macOS, Windows y Linux).
Codespaces ofrece el mismo entorno para todos sin configuración local.

Alternativas: Docker Desktop local, WSL2, máquina virtual.

Tradeoff: depende de conectividad y de los free-tier hours disponibles (60 hs/mes por cuenta).
Con Docker local se trabaja offline y sin límite de tiempo.

Resultado: Codespaces para las clases, Docker local como fallback documentado en el README.
### 005 - Identidad y credenciales en el lab

Decision: usar roles con STS en lugar de access keys de larga duracion para acceso entre servicios.

Contexto: las access keys no expiran y si se filtran dan acceso indefinido. Los roles con STS generan credenciales temporales (15 min a 12 hs) con trazabilidad.

Alternativas: access keys rotadas manualmente, vault/secret manager.

Tradeoff: asumir un rol requiere que el servicio tenga permiso de sts:AssumeRole y que el rol tenga un trust policy correcto. Mas configuracion inicial, menos riesgo.

Resultado: app-role con inline policy de privilegio minimo sobre course-data-raw.

Nota sobre el entorno: LocalStack Community no hace enforcement de politicas. Un Deny explicito no bloquea la llamada (se verifico en el paso 9). Para ver enforcement real se necesita LocalStack Pro o AWS real.
### 006 - Instance profile en lugar de access keys en la instancia

Decision: usar instance profile (rol via IMDSv2) en lugar de access keys hardcodeadas en la VM.

Contexto: una instancia que necesita leer S3 puede acceder por dos caminos: (a) access keys guardadas en disco, o (b) un rol asociado via instance profile que devuelve credenciales temporales por IMDSv2.

Tradeoff: la opcion (a) es mas directa pero deja claves de larga duracion en disco; si la instancia se compromete o se snapshotea, esas claves quedan expuestas. La opcion (b) requiere setup inicial pero las credenciales rotan automaticamente y nunca tocan disco.

Resultado: instance profile 'app-instance-profile' con el rol 'app-role' del lab 04, adjuntado a una instancia EC2 con security group web-sg (puertos 80 y 22).

### 007 - course-data-lake como fuente durable del modulo

Decision: separar 'course-data-raw' (demo IAM del lab 04) de 'course-data-lake' (fuente de verdad de datos reales del curso). El segundo nace con BPA, encryption y versioning ON, y bucket policy que restringe lectura al instance role de la app.

Contexto: necesitamos un lugar durable para Olist + GitHub Archive que sobreviva al ciclo de vida de cada lab. Mezclar con el bucket de demo IAM enmascara el proposito de cada uno.

Tradeoff: dos buckets en lugar de uno. A favor: separacion clara de intencion, escalable a futuras clases (Analytics consume directo desde la lake).

Resultado: course-data-lake con versioning + BPA + SSE + bucket policy desde el dia 1.

