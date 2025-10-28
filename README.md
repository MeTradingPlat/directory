# Directory - Eureka Server

Servidor de registro y descubrimiento de servicios (Service Registry) para la plataforma **MeTradingPlat**. Implementa **Netflix Eureka Server** para permitir que los microservicios se descubran y comuniquen entre sí de forma dinámica.

## Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Arquitectura](#arquitectura)
- [Tecnologías](#tecnologías)
- [Instalación](#instalación)
- [Configuración](#configuración)
- [Uso](#uso)
- [Endpoints](#endpoints)
- [Microservicios Registrados](#microservicios-registrados)
- [Monitoreo](#monitoreo)
- [Desarrollo](#desarrollo)

---

## Descripción General

**Directory** es el componente central de la arquitectura de microservicios, actuando como **Service Registry** mediante Netflix Eureka Server. Permite que los microservicios:

- Se registren automáticamente al iniciar
- Descubran otros servicios sin conocer sus URLs
- Se comuniquen de forma dinámica y resiliente
- Implementen balanceo de carga del lado del cliente

### Características Principales

- **Service Discovery**: Registro y descubrimiento automático de microservicios
- **Health Checking**: Verificación periódica del estado de los servicios
- **Virtual Threads**: Alta concurrencia con Java 21
- **Dashboard Web**: Interfaz gráfica para monitoreo
- **Auto-limpieza**: Eliminación automática de instancias caídas

---

## Arquitectura

### Patrón: Service Registry

```
┌─────────────────────────────────────────────────────────────┐
│                     EUREKA SERVER                           │
│                    (directory:8761)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           SERVICE REGISTRY                           │  │
│  │  ┌────────────────────┬──────────────────────────┐   │  │
│  │  │  Service Name      │  Instances               │   │  │
│  │  ├────────────────────┼──────────────────────────┤   │  │
│  │  │  GATEWAY           │  localhost:8080          │   │  │
│  │  │  GESTION-ESCANERES │  localhost:8081          │   │  │
│  │  └────────────────────┴──────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           HEALTH MONITORING                          │  │
│  │  - Heartbeat cada 30 segundos                        │  │
│  │  - Eliminación si no responde en 90 segundos         │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
             ▲                           ▲
             │                           │
             │ Register/Heartbeat        │ Register/Heartbeat
             │                           │
    ┌────────┴─────────┐        ┌───────┴────────┐
    │     GATEWAY      │        │ GESTION-       │
    │   (Port 8080)    │        │ ESCANERES      │
    │                  │        │ (Port 8081)    │
    └──────────────────┘        └────────────────┘
```

### Flujo de Registro

1. **Microservicio arranca** → Se conecta a Eureka Server
2. **Registro inicial** → Envía metadata (nombre, IP, puerto, health URL)
3. **Heartbeat** → Envía señal cada 30 segundos
4. **Health Check** → Eureka verifica estado del servicio
5. **Auto-limpieza** → Elimina servicios que no responden en 90s

---

## Tecnologías

| Tecnología | Versión | Descripción |
|-----------|---------|-------------|
| **Java** | 21 | Lenguaje principal (Virtual Threads) |
| **Spring Boot** | 3.4.11 | Framework base |
| **Spring Cloud** | 2024.0.0 | Suite de microservicios |
| **Netflix Eureka Server** | Latest | Service Discovery |
| **Spring Actuator** | 3.4.11 | Monitoreo y métricas |
| **Maven** | 3.9+ | Gestión de dependencias |

---

## Instalación

### Prerrequisitos

- **Java 21+** (con soporte para Virtual Threads)
- **Maven 3.9+**

### Clonar el Repositorio

```bash
git clone https://github.com/tu-org/metradingplat.git
cd metradingplat/directory
```

### Compilar el Proyecto

```bash
./mvnw clean install
```

---

## Configuración

### Archivo: `application.properties`

```properties
# Nombre de la aplicación
spring.application.name=directory

# Puerto del servidor Eureka
server.port=8761

# Configuración de Eureka Server
eureka.instance.hostname=directory
eureka.instance.prefer-ip-address=true

# NO registrarse a sí mismo (es el servidor de registro)
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

# Virtual Threads para alta concurrencia
spring.threads.virtual.enabled=true
```

### Configuración Avanzada

Para personalizar el comportamiento de Eureka:

```properties
# Tiempo de espera antes de eliminar un servicio inactivo (ms)
eureka.server.eviction-interval-timer-in-ms=60000

# Desactivar auto-preservación en desarrollo
eureka.server.enable-self-preservation=false

# Umbral de renovación
eureka.server.renewal-percent-threshold=0.85
```

### Ejecutar el Servicio

```bash
./mvnw spring-boot:run
```

El servidor estará disponible en: `http://localhost:8761`

---

## Uso

### Dashboard Web

Accede al dashboard de Eureka en tu navegador:

```
http://localhost:8761
```

El dashboard muestra:
- Servicios registrados actualmente
- Número de instancias de cada servicio
- Estado de salud de cada instancia
- Última renovación (heartbeat)
- Metadata de cada servicio

### Registrar un Microservicio Cliente

Los microservicios se registran agregando esta dependencia:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

Y configurando en `application.properties`:

```properties
# Nombre del servicio (aparecerá en Eureka)
spring.application.name=mi-servicio

# URL del servidor Eureka
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/

# Preferir IP en lugar de hostname
eureka.instance.prefer-ip-address=true
```

---

## Endpoints

### Dashboard y API REST

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/` | GET | Dashboard web de Eureka |
| `/eureka/apps` | GET | Lista de aplicaciones registradas (XML) |
| `/eureka/apps/{appName}` | GET | Información de una aplicación específica |
| `/eureka/apps/{appName}/{instanceId}` | GET | Información de una instancia específica |
| `/actuator/health` | GET | Estado de salud del servidor Eureka |
| `/actuator/info` | GET | Información del servidor |

### Ejemplo de Respuesta: `/eureka/apps`

```xml
<applications>
  <application>
    <name>GATEWAY</name>
    <instance>
      <instanceId>localhost:gateway:8080</instanceId>
      <hostName>localhost</hostName>
      <app>GATEWAY</app>
      <ipAddr>192.168.1.100</ipAddr>
      <status>UP</status>
      <port enabled="true">8080</port>
      <healthCheckUrl>http://localhost:8080/actuator/health</healthCheckUrl>
    </instance>
  </application>
  <application>
    <name>GESTION-ESCANERES</name>
    <instance>
      <instanceId>localhost:gestion-escaneres:8081</instanceId>
      <hostName>localhost</hostName>
      <app>GESTION-ESCANERES</app>
      <ipAddr>192.168.1.100</ipAddr>
      <status>UP</status>
      <port enabled="true">8081</port>
      <healthCheckUrl>http://localhost:8081/actuator/health</healthCheckUrl>
    </instance>
  </application>
</applications>
```

---

## Microservicios Registrados

En la arquitectura de **MeTradingPlat**, los siguientes servicios se registran con Eureka:

| Servicio | Puerto | Descripción |
|----------|--------|-------------|
| **gateway** | 8080 | API Gateway - Punto de entrada único |
| **gestion-escaneres** | 8081 | Gestión de escaners de mercado |

### Flujo de Comunicación

```
Cliente HTTP
    │
    ▼
┌─────────────┐
│   GATEWAY   │ ← Consulta a Eureka: "¿Dónde está gestion-escaneres?"
│  (Port 8080)│
└─────────────┘
    │
    │ Eureka responde: "localhost:8081"
    │
    ▼
┌─────────────────┐
│ GESTION-        │
│ ESCANERES       │
│ (Port 8081)     │
└─────────────────┘
```

---

## Monitoreo

### Health Check

Verificar el estado del servidor Eureka:

```bash
curl http://localhost:8761/actuator/health
```

**Respuesta:**
```json
{
  "status": "UP"
}
```

### Métricas

Obtener métricas del servidor:

```bash
curl http://localhost:8761/actuator/metrics
```

### Logs

Los logs de Eureka muestran:
- Registros de nuevos servicios
- Heartbeats recibidos
- Servicios eliminados por timeout
- Errores de comunicación

---

## Desarrollo

### Ejecutar en Modo Desarrollo

```bash
./mvnw spring-boot:run
```

### Compilar JAR para Producción

```bash
./mvnw clean package
java -jar target/directory-0.0.1-SNAPSHOT.jar
```

### Estructura del Proyecto

```
directory/
│
├── src/main/java/com/metradingplat/directory/
│   └── DirectoryApplication.java          # Clase principal con @EnableEurekaServer
│
├── src/main/resources/
│   └── application.properties             # Configuración de Eureka Server
│
└── pom.xml                                # Dependencias Maven
```

### Código Principal

```java
@SpringBootApplication
@EnableEurekaServer  // Habilita Eureka Server
public class DirectoryApplication {
    public static void main(String[] args) {
        SpringApplication.run(DirectoryApplication.class, args);
    }
}
```

---

## Solución de Problemas

### Problema: Servicios no se registran

**Causa:** URL incorrecta de Eureka en el cliente

**Solución:**
```properties
# En el microservicio cliente
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

### Problema: Auto-preservación activada

**Síntoma:** Mensaje "EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT"

**Causa:** Menos del 85% de instancias renovaron su heartbeat

**Solución (solo desarrollo):**
```properties
eureka.server.enable-self-preservation=false
```

### Problema: Servicios duplicados

**Causa:** Instancias con el mismo `instanceId`

**Solución:**
```properties
# Hacer instanceId único
eureka.instance.instance-id=${spring.application.name}:${random.value}
```

---

## Configuración en Producción

### Variables de Entorno

```bash
export SERVER_PORT=8761
export EUREKA_HOSTNAME=eureka.metradingplat.com
```

### Docker

```dockerfile
FROM openjdk:21-jdk-slim
WORKDIR /app
COPY target/directory-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8761
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Docker Compose

```yaml
version: '3.8'
services:
  eureka-server:
    build: ./directory
    ports:
      - "8761:8761"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Mejores Prácticas

1. **Alta Disponibilidad**: Ejecutar múltiples instancias de Eureka Server en producción
2. **Seguridad**: Proteger el dashboard con autenticación
3. **Monitoreo**: Integrar con sistemas de monitoreo (Prometheus, Grafana)
4. **Logs**: Centralizar logs con ELK Stack
5. **Health Checks**: Configurar health endpoints en todos los clientes

---

## Recursos Adicionales

- [Spring Cloud Netflix Eureka](https://spring.io/projects/spring-cloud-netflix)
- [Eureka Wiki](https://github.com/Netflix/eureka/wiki)
- [Microservices Patterns](https://microservices.io/patterns/service-registry.html)

---

## Licencia

Copyright (c) 2025 MeTradingPlat. Todos los derechos reservados.

---

## Contacto

Para soporte o consultas: [contacto@metradingplat.com](mailto:contacto@metradingplat.com)

---

## Changelog

### v0.0.1-SNAPSHOT (Actual)
- Implementación inicial de Eureka Server
- Virtual Threads (Java 21)
- Dashboard web habilitado
- Actuator endpoints configurados
