
# Informe de Seguridad Accent GeoGuessr

## Estado de la Aplicación

La instancia públicamente accesible de la aplicación en accentgeoguessr-classroom.streamlit.app ha sido retirada temporalmente. Esto permite implementar las medidas descritas en este informe sin exponer a los usuarios a un riesgo activo durante la limpieza. Los resultados del informe reflejan el estado anterior a la retirada. Una vez aplicadas las correcciones, la aplicación puede volver a publicarse con una seguridad mejorada.

## Objetivo y Alcance del Análisis

Se analizó la aplicación del repositorio público de GitHub github.com/quentinrauschenbach/accent_geoguessr. Se revisó todo el código contenido en el repositorio, con especial atención en los dos archivos principales app.py y app_local.py, ya que contienen toda la lógica de la aplicación. Además se consideraron las dependencias en requirements.txt y el historial de commits. Complementariamente, la instancia en vivo fue visitada manualmente y revisada en ambas áreas de la aplicación: la vista normal del estudiante en https://accentgeoguessr-classroom.streamlit.app/ y el panel del profesor en https://accentgeoguessr-classroom.streamlit.app/?role=teacher. El análisis cubrió las respuestas del servidor, las cookies establecidas y la configuración visible de ambas áreas.

## 1. Resumen

La aplicación guarda todo el estado del juego en un almacenamiento global en el servidor, y la autenticación existe prácticamente solo para el profesor, implementada de forma débil. Para un juego puramente escolar la mayoría de los hallazgos sería tolerable. Sin embargo, la instancia funcionaba públicamente en internet, lo que hace los hallazgos críticos realmente explotables. El riesgo principal no está en vulnerabilidades clásicas de inyección sino en lógica y configuración: toma de control del rol de profesor, manipulación del estado del juego y phishing de los estudiantes mediante un código QR manipulado. Adicionalmente se observaron debilidades en la capa de entrega al inspeccionar las respuestas del servidor.

## 2. Hallazgos Críticos

### 2.1 Contraseña en Texto Plano en un Repositorio Público

En app.py la contraseña del profesor es legible línea por línea:

```python
TEACHER_PASSWORD = "0712"
```

La contraseña es públicamente visible. Cualquiera que conozca el nombre del repositorio puede tomar el control de la función de profesor de inmediato. Además consta de cuatro dígitos que parecen mucho una fecha, por lo que incluso un cambio sería fácil de adivinar. Cambiar solo el archivo no basta, porque el valor antiguo permanece conservado de forma permanente en el historial de commits. Herramientas como BFG Repo Cleaner o git filter repo sirven para limpiar el historial. En el caso más simple, el valor antiguo se considera quemado definitivamente y no se vuelve a usar jamás.

Recomendación: Mover la contraseña a Streamlit Secrets, cargándola mediante st.secrets. El archivo secrets.toml pertenece al gitignore. La nueva contraseña debe contener al menos doce caracteres aleatorios sin patrón reconocible.

### 2.2 Sin Protección Contra el Adivinamiento de la Contraseña

El inicio de sesión del profesor compara la contraseña directamente en cada intento, sin contador, sin retardo, sin bloqueo. Un código de cuatro dígitos comprende diez mil combinaciones y puede agotarse con un script simple en menos de un minuto. Streamlit no limita las peticiones por defecto.

Recomendación: Un contador de intentos por sesión y dirección, con un bloqueo de varios minutos tras cinco intentos fallidos. Más robusto sería una barrera real delante de la aplicación, como un proxy inverso con autenticación básica en la ruta del profesor. Secrets más limitación de tasa son no obstante suficientes para este propósito.

## 3. Riesgos Altos

### 3.1 URL de Acceso Manipulable Mediante la Cabecera Host

```python
host_url = st.context.headers.get("host", "localhost:8501")
STUDENT_JOIN_URL = f"https://{host_url}"
```

La cabecera Host la envía el cliente y por tanto la controla el atacante. Si alguien abre la página con una cabecera Host falsificada, la aplicación genera un código QR que apunta al dominio del atacante. Si ese código se muestra en el proyector, los estudiantes caen en una página clonada. Este escenario se conoce como web cache poisoning o host header injection.

Recomendación: Guardar la URL pública de forma fija, por ejemplo como secreto de Streamlit, en lugar de deducirla de la petición. Alternativamente una lista blanca de nombres de host permitidos y un rechazo en caso de desviación.

### 3.2 Estado del Juego Manipulable

Todo el estado del juego reside en un único diccionario compartido mediante st.cache_resource. De ahí resultan tres problemas, inofensivos por separado pero sustanciales en conjunto.

Primero, los apodos no están protegidos. Quien introduce el mismo nombre que un compañero escribe entradas bajo esa identidad en la tabla all_guesses, porque el nombre solo sirve como clave. Así pueden estropearse resultados ajenos o apropiarse puntos ajenos.

Segundo, el bloqueo de apuesta existe solo en session_state. El botón LOCK IN GUESS establece una bandera en la sesión del navegador. Al abrir una pestaña de incógnito nueva, la bandera desaparece y se puede apostar de nuevo. Nada en el servidor impide múltiples apuestas por persona y ronda.

Tercero, falta un plazo. Las apuestas siguen entrando en la puntuación aunque la solución ya sea visible en el proyector, mientras la ronda siga activa en el contador. Quien lee la posición verdadera de la pantalla y abre rápidamente una pestaña nueva obtiene los puntos completos.

Recomendación: Un token de acceso aleatorio por estudiante, guardado con el nombre y enviado con cada apuesta. En el servidor, comprobar si ya existe una entrada para la combinación de ronda y nombre y descartar las adicionales. Cuando show_leaderboard sea verdadero, dejar de aceptar apuestas para la ronda actual. Cada una de estas medidas ocupa pocas líneas.

### 3.3 Verificación de Rol Presumiblemente Solo en la Entrada

El acceso a la vista del profesor funciona mediante el parámetro de consulta role=teacher y la contraseña. Del código visible no quedó completamente claro si la autenticación se vuelve a comprobar antes de cada bloque del profesor o solo en la primera carga. Un error frecuente en Streamlit es que tras un rerun o un session_state manipulado las áreas protegidas se vuelven accesibles sin nueva comprobación. Por tanto debe verificarse si acciones como la subida, el cambio de ronda o el reinicio dependen solo de que el bloque de código del profesor se ejecute, y no de una variable comprobada explícitamente como st.session_state.authenticated.

Recomendación: Una única variable is_teacher fijada en session_state, establecida solo tras una comparación de contraseña exitosa, y cada bloque del profesor comienza comprobando esa variable.

## 4. Riesgos Medios

### 4.1 Subida de Archivos Sin Endurecimiento Visible

La subida de archivos de audio y vídeo a la carpeta clips no muestra validación de nombre, tamaño ni tipo de contenido en el código revisado. Tres peligros concretos: Un nombre de archivo con componentes de ruta puede acabar fuera de la carpeta destino si el nombre se usa sin filtrar. Sin límite de tamaño una sola subida puede llenar el almacenamiento libre de Streamlit Cloud, unos un gigabyte, tras lo cual la aplicación queda inaccesible para todos. Sin comprobación de extensión, archivos arbitrarios acaban en el servidor.

Recomendación: Regenerar el nombre de archivo guardado en el servidor, por ejemplo con uuid, conservando el nombre original solo como texto de visualización. Un límite duro de veinte a cincuenta megabytes. Restringir las extensiones estrictamente a mp3, m4a, wav y mp4, idealmente comprobando también el tipo mime. Ofrecer la subida exclusivamente tras la autenticación del profesor.

### 4.2 Funciones de Control Globales Alcanzables por los Estudiantes

Aunque la interfaz muestra las funciones de control solo al profesor, esto es un problema puramente visual. Las aplicaciones Streamlit no tienen concepto de roles en el servidor; cada sesión ejecuta el mismo código. Si acciones como cambios de ronda o reinicios del juego dependen de la ruta de visualización en lugar de la autenticación, un estudiante con conocimientos técnicos puede activarlas. Esta comprobación pertenece al mismo esfuerzo que el punto 3.3.

## 5. Riesgos Bajos

El archivo requirements.txt no contiene versiones fijadas. Aceptable para un proyecto escolar, pero fijar versiones evita sorpresas desagradables con actualizaciones de Streamlit.

Los apodos no tienen límite de longitud. Nombres muy largos deforman la tabla de clasificación; treinta caracteres más una restricción a caracteres sensatos es una medida de una línea.

app_local.py genera el código QR con la dirección local mediante http sin cifrar. Tolerable en una red escolar protegida, pero cualquiera en la misma red ve el tráfico y puede participar en el juego. La variante debería desactivarse para el funcionamiento fuera del aula.

La memoria del juego vive solo en la RAM; un reinicio borra todo. No es un problema de seguridad, pero explica la ausencia de salvaguardas en el servidor y sería un argumento para mover el estado a un archivo sqlite o una pequeña base de datos si fuera necesario.

## 6. Orden Priorizada de Corrección

Primero la contraseña. Usar secrets, elegir una contraseña fuerte, considerar el valor antiguo como quemado definitivamente y limpiar el historial si es posible.

Después el endurecimiento de la autenticación: limitación de tasa en el inicio de sesión y una variable is_teacher comprobada consistentemente antes de cada bloque del profesor, incluidos subida y reinicio.

Luego guardar la URL de acceso de forma fija en lugar de construirla desde la cabecera host.

A continuación la integridad del juego: tokens de acceso, comprobación de duplicados por ronda y nombre, y un corte de apuestas desde el momento de la revelación.

Después el endurecimiento de la subida con nombres de archivo regenerados, límite de tamaño y comprobación de extensiones.

Al final los puntos menores: fijado de versiones, límites de longitud de nombres y conciencia del funcionamiento en texto plano en la red local.

## 7. Observaciones Sobre la Capa de Entrega

Además del código fuente, la instancia en vivo fue visitada, en particular las respuestas que el servidor envía al navegador y las cookies establecidas en ambas áreas de la aplicación, el panel del estudiante y el panel del profesor. Esta capa queda fuera del control de la aplicación en Streamlit Community Cloud, por lo que muchos de los siguientes puntos deben entenderse como documentados y del lado de la plataforma.

### 7.1 Content Security Policy, No Presente

No se observó ninguna Content Security Policy. Una CSP es la línea de defensa más importante contra la inyección de scripts ajenos, es decir contra cross site scripting. En una aplicación Streamlit es especialmente relevante porque la interfaz consta de componentes cargados dinámicamente, y un atacante que logre inyectar contenido tendría vía libre sin una CSP.

Recomendación: Establecer una Content Security Policy mediante la cabecera del mismo nombre. Un comienzo sensato restringe default-src a self y permite explícitamente solo las fuentes que la aplicación realmente necesita, es decir las teselas de mapa de OpenStreetMap y las conexiones propias de Streamlit. Empezar mejor en modo report only, observar la consola en busca de violaciones y endurecer la política después.

### 7.2 X-Frame-Options y Clickjacking, No Presente

La página puede incrustarse en un marco ajeno, por lo que falta la protección de marcos. Esto permite clickjacking, la superposición de la página real con una superficie engañosa mediante la cual un usuario activa acciones sin saberlo. En el contexto escolar el riesgo es bajo, pero la corrección casi no cuesta nada.

Recomendación: O bien la cabecera clásica X-Frame-Options con el valor DENY o SAMEORIGIN, o más moderno la directiva frame-ancestors none dentro de la Content Security Policy del punto 7.1.

### 7.3 Redirección y Falta de Strict Transport Security

La primera redirección de http a https va a un host distinto. Como consecuencia la cabecera Strict Transport Security se descarta en el primer contacto, porque HSTS solo es válido sobre una conexión cifrada al mismo host. Además se observó que las respuestas de la instancia no contienen ninguna cabecera Strict Transport Security.

Recomendación: Reconstruir la cadena de modo que el primer salto a https ocurra en el mismo dominio, con posibles redirecciones posteriores. Hay poca influencia posible en Streamlit Community Cloud; el camino lleva por un dominio propio con un proxy delante.

### 7.4 Subresource Integrity, No Presente

Los scripts externos se cargan cifrados pero sin verificación de integridad. SRI significa que el atributo HTML lleva un hash del script esperado y el navegador rechaza el archivo si ha sido alterado. La relevancia práctica aquí es baja porque todo se transmite al menos cifrado.

Recomendación: Añadir los atributos integrity y crossorigin a todas las etiquetas script y link externas. Como una aplicación Streamlit no escribe estas etiquetas ella misma, la CSP del punto 7.1 ayuda más aquí.

### 7.5 Referrer Policy, No Establecida

La cabecera falta en las respuestas observadas. Al hacer clic en enlaces externos puede transmitirse así la dirección completa de destino incluidos los parámetros a la página ajena.

Recomendación: Establecer la cabecera Referrer-Policy en strict-origin-when-cross-origin.

### 7.6 Cross Origin Policies, No Establecidas

Faltan las tres cabeceras Cross Origin Embedder Policy, Cross Origin Opener Policy y Cross Origin Resource Policy. Evitan entre otras cosas que páginas ajenas construyan relaciones de ventana con la aplicación o incrusten sus recursos.

Recomendación: same-origin para la política de opener, require-corp o credentialless para la de embedder, y same-origin para la de resource. Probar primero las configuraciones en modo observación, porque los mapas de folium cargan teselas externas.

### 7.7 Cookies, un Cuadro Mixto

Las cookies de sesión propias de Streamlit están correctamente protegidas con Secure, HttpOnly y SameSite. Llama la atención en cambio una cookie llamada proxy-tracking-id establecida sin HttpOnly y sin Secure. Procede de la infraestructura de la plataforma y no puede controlarse desde la propia aplicación. Como esta cookie sirve más para el seguimiento de conexiones que para llevar una sesión real, el daño concreto es manejable. No obstante es incómoda la combinación con la falta de Content Security Policy del punto 7.1, porque una CSP ausente reduce el umbral de inyección existiendo al mismo tiempo una cookie expuesta. El camino de remedio es el mismo que allí: un dominio propio con un proxy delante donde se controlen directamente todas las cabeceras establecidas.

### 7.8 Otras Observaciones Positivas

No hay permisos CORS abiertos, y la página principal contiene la cabecera nosniff, que evita el rastreo de tipos mime. Llama la atención que esta cabecera falta en las rutas de recursos estáticos bajo build y assets. El riesgo es bajo. Con un traslado posterior detrás de un proxy propio, la cabecera pertenece a cada respuesta, más simplemente mediante una configuración global del proxy.

### 7.9 Información del Servidor

Al inspeccionar los datos de conexión quedó visible que detrás de la aplicación están Google Cloud, una red de distribución de contenido y un Nginx, siendo visible el número de versión del Nginx. Semejante información permite a un atacante buscar específicamente vulnerabilidades de exactamente esa versión. Como la infraestructura del servidor en Streamlit Community Cloud no es controlable, el punto queda como nota documentada.

### 7.10 Lo Que No Se Observó

No se notaron puntos finales de depuración abiertos, ni listados de directorios, ni mensajes de error visibles con detalles internos, ni contraseñas transmitidas sin cifrar, ni claves privadas en las respuestas. Las superficies de ataque relevantes de esta aplicación no son de todos modos las vulnerabilidades web clásicas, porque no hay base de datos ni consulta en el servidor que traduzca entradas de usuario a SQL o comandos de shell. Una evaluación técnica profunda cobra sentido solo cuando se añade una base de datos o una autenticación real con tokens.

## 8. Evaluación General

El cuadro es de dos mitades. Dentro del código de la aplicación los temas críticos son la contraseña, la limitación de tasa y la manipulación del estado. En la capa de entrega faltan sobre todo la Content Security Policy y la protección de marcos, y una cookie de seguimiento de la plataforma está insuficientemente protegida. Tras la contraseña y la autenticación, la tercera medida es la Content Security Policy junto con frame-ancestors, después la URL de acceso fija, la integridad del juego y la subida. La base de código está sólidamente estructurada para un proyecto de fin de semana creado con ayuda de IA. La retirada temporal de la instancia es el paso correcto para implementar las correcciones con calma antes de que la aplicación vuelva a ser públicamente accesible.
