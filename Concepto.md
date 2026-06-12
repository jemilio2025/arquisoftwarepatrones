# Patrón Gatekeeper aplicado a AgentWatch (Módulo de Seguridad)

## 1. Problema

En aplicaciones modernas desplegadas en la nube, los servicios suelen estar expuestos a usuarios, aplicaciones externas y múltiples sistemas conectados mediante APIs. Esta exposición incrementa la superficie de ataque y genera riesgos relacionados con accesos no autorizados, filtración de datos, robo de credenciales y manipulación de solicitudes.

El problema se vuelve más complejo en plataformas basadas en inteligencia artificial, donde los usuarios interactúan mediante lenguaje natural. En estos casos, un atacante puede intentar engañar al sistema utilizando instrucciones ocultas dentro de un mensaje aparentemente legítimo, lo que se conoce como *prompt injection*.

En AgentWatch, los principales riesgos se encuentran en cuatro áreas:

* Acceso indebido a recursos mediante autenticación insuficiente.
* Fuga de información entre distintos clientes o tenants.
* Exposición de credenciales utilizadas por servicios internos.
* Manipulación de agentes de IA mediante prompts maliciosos.

El desafío consiste en impedir que solicitudes peligrosas lleguen a los componentes internos encargados de procesar datos y ejecutar acciones.

## 2. Solución

El patrón Gatekeeper propone insertar un componente intermediario entre los usuarios externos y los servicios internos. Este componente recibe todas las solicitudes, las inspecciona y decide si pueden continuar o deben ser bloqueadas.

La idea central es separar la lógica de validación de la lógica de negocio. De esta manera, los sistemas internos nunca quedan expuestos directamente.

Microsoft define el Gatekeeper como un mecanismo de protección para aplicaciones y servicios. Sin embargo, en entornos actuales el patrón ha evolucionado y también se utiliza para aplicar políticas de seguridad, control de acceso, auditoría y filtrado inteligente de contenido.

En AgentWatch, el patrón se implementa en cuatro niveles de protección.

### Gatekeeper de autenticación y autorización (RF13)

La primera barrera controla quién puede ingresar al sistema.

Cada solicitud pasa por un middleware de FastAPI encargado de:

* Validar tokens JWT.
* Verificar permisos según el rol del usuario.
* Aplicar límites de solicitudes para evitar abusos.

Herramientas utilizadas:

* python-jose o python-jwt para la gestión de JWT.
* slowapi para rate limiting.
* Supabase Auth como proveedor de autenticación con OAuth.

En esta capa, los servicios internos solo reciben solicitudes previamente verificadas.

### Gatekeeper de aislamiento multi-tenant (RF14)

La segunda barrera protege los datos.

AgentWatch está diseñado para atender múltiples organizaciones. Cada cliente debe acceder únicamente a su propia información.

Para garantizarlo se utiliza PostgreSQL Row Level Security (RLS), que filtra automáticamente los registros según el tenant asociado al usuario.

Esto evita errores humanos, ya que el desarrollador no necesita recordar aplicar filtros en cada consulta.

A nivel de infraestructura, Kubernetes complementa esta protección mediante:

* Namespaces separados.
* Network Policies que restringen la comunicación entre entornos.

Herramientas utilizadas:

* PostgreSQL con Row Level Security.
* Kubernetes NetworkPolicy.
* Pytest para pruebas de aislamiento entre tenants.

### Gatekeeper de gestión de credenciales (RF15)

La tercera barrera protege secretos y credenciales.

Las API Keys, contraseñas y tokens no deben almacenarse dentro del código ni exponerse en registros del sistema.

AgentWatch utiliza Azure Key Vault como intermediario entre la aplicación y los secretos.

Cuando un servicio necesita una credencial:

1. Solicita acceso al Vault.
2. El Vault verifica permisos.
3. Registra la operación.
4. Entrega el secreto de forma segura.

Herramientas utilizadas:

* SDK azure-keyvault-secrets.
* Azure Managed Identity para autenticación automática.

Gracias a este enfoque, las credenciales nunca quedan expuestas directamente.

### Gatekeeper contra Prompt Injection (RF16)

La cuarta barrera está orientada a sistemas basados en inteligencia artificial.

Los modelos de lenguaje reciben instrucciones escritas por los usuarios. Un atacante puede aprovechar este mecanismo para intentar modificar el comportamiento del agente mediante mensajes diseñados para ignorar reglas o revelar información sensible.

Antes de que un prompt llegue al modelo, AgentWatch incorpora un Gatekeeper especializado que:

* Analiza el contenido.
* Detecta patrones de ataque.
* Clasifica el nivel de riesgo.
* Bloquea o pone en cuarentena solicitudes sospechosas.

Herramientas utilizadas:

* Garak para pruebas de seguridad en LLMs.
* LLM Guard para sanitización de entradas.
* OWASP ZAP y Burp Suite para pruebas de penetración.
* Metasploit y Nmap para simulaciones de ataque.

Este mecanismo reduce significativamente el riesgo de manipulación de los agentes inteligentes.

## 3. Casos de Aplicación

### Plataformas SaaS Multi-Tenant

Empresas que ofrecen software como servicio a múltiples clientes utilizan Gatekeeper para asegurar que cada organización acceda únicamente a sus propios datos.

Ejemplos:

* Sistemas ERP en la nube.
* Plataformas CRM.
* Herramientas de gestión empresarial.

### Servicios Financieros

Bancos y fintech emplean Gatekeepers para validar identidades, aplicar controles de acceso y proteger información financiera sensible antes de permitir cualquier operación.

Ejemplos:

* Banca digital.
* Pasarelas de pago.
* Sistemas de inversión en línea.

### Plataformas de Salud

Hospitales y sistemas de historia clínica electrónica requieren controlar estrictamente quién puede acceder a información médica.

El Gatekeeper permite verificar permisos antes de consultar datos de pacientes.

### Aplicaciones basadas en Inteligencia Artificial

Asistentes virtuales, agentes autónomos y plataformas de IA generativa utilizan Gatekeepers para analizar prompts y detectar intentos de manipulación.

Ejemplos:

* Chatbots corporativos.
* Copilots empresariales.
* Sistemas de atención al cliente basados en LLMs.

### AgentWatch

En AgentWatch, el patrón Gatekeeper es uno de los elementos centrales de la arquitectura de seguridad.

Se aplica en:

* Autenticación y control de acceso.
* Separación de datos entre organizaciones.
* Protección de credenciales.
* Defensa frente a ataques de prompt injection.

Gracias a esta implementación en múltiples capas, el sistema reduce el riesgo de accesos no autorizados, filtraciones de información y manipulación de agentes inteligentes.

## 4. Aporte Conceptual

La documentación de Microsoft presenta el patrón Gatekeeper principalmente para APIs y servicios tradicionales. Sin embargo, los sistemas de inteligencia artificial introducen un nuevo tipo de amenaza: los ataques semánticos.

En una aplicación convencional, las validaciones se realizan sobre estructuras definidas, como campos, formatos o tipos de datos. En cambio, un sistema basado en LLM recibe texto libre, donde una instrucción maliciosa puede estar oculta dentro de una conversación aparentemente normal.

Por esta razón, el patrón Gatekeeper debe evolucionar e incorporar mecanismos capaces de interpretar el significado del contenido.

Una tendencia actual es el uso de un modelo de lenguaje como evaluador de seguridad, conocido como *LLM-as-Judge* o *Guardrail LLM*. En este enfoque, un modelo analiza el prompt recibido y determina si representa un riesgo antes de permitir que llegue al agente principal.

Esta adaptación amplía el alcance del patrón Gatekeeper y lo convierte en una pieza clave para proteger aplicaciones basadas en IA generativa.


Esto exige que el Gatekeeper de RF16 incorpore un componente de razonamiento — literalmente otro LLM actúa como guardia evaluando si el input del usuario es malicioso.

Esto se llama "LLM-as-judge" o "guardrail LLM" y es la extensión natural del patrón Gatekeeper al mundo de la IA generativa.
