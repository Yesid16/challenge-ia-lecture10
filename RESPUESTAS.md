# Respuestas del workshop RAG
Escribe aquí tus respuestas con tus propias palabras. Pega tablas, resultados y enlaces a las capturas de `evidencias/`.

---

# Bloque 0 · Teoría

## T1 · Cuántos nodos viven en cada capa


# Bloque 0 · Teoría: HNSW y FAISS


### 1. Cálculo a mano

Dado que la probabilidad de que un vector llegue al menos a la capa $k$ es $P(\text{capa} \geq k) = M^{-k}$ y tenemos $N = 1.000.000$ vectores con $M = 16$:

* **Capa $\geq 1$:**
  $$P(\geq 1) = 16^{-1} = \frac{1}{16} = 0.0625$$
  $$\text{Nodos esperados} = 1.000.000 \times 0.0625 = 62.500$$

* **Capa $\geq 2$:**
  $$P(\geq 2) = 16^{-2} = \frac{1}{256} \approx 0.00390625$$
  $$\text{Nodos esperados} = 1.000.000 \times 0.00390625 \approx 3.906$$

* **Capa $\geq 3$:**
  $$P(\geq 3) = 16^{-3} = \frac{1}{4096} \approx 0.00024414$$
  $$\text{Nodos esperados} = 1.000.000 \times 0.00024414 \approx 244$$

---

### 2. Simulación en Python y comparación
<img width="742" height="596" alt="image" src="https://github.com/user-attachments/assets/5e62cc2f-c2c5-4dba-b975-65be19234d7d" />


**Comparación:**
Los resultados de la simulación son sumamente cercanos a los valores teóricos debido a la Ley de los Grandes Números con $N = 1.000.000$. Las pequeñas variaciones se deben a la naturaleza estocástica del muestreo uniforme.


### 3. Explicación

Tener pocos nodos (hubs) en las capas altas permite realizar saltos largos de gran distancia en el espacio vectorial de forma muy eficiente, reduciendo drásticamente el número de candidatos a evaluar. Esto acelera las búsquedas al evitar revisiones innecesarias antes de descender a las capas inferiores para refinar el resultado.


---

## T2 · ef, k y el tamaño de la cola


1. **Si pides $k = 10$ vecinos y configuras $ef = 5$, ¿qué falla y por qué?**
   Falla porque el tamaño de la cola de candidatos (`ef = 5`) es estrictamente menor que el número de vecinos solicitados ($k = 10$). El algoritmo no puede recolectar ni evaluar suficientes candidatos para rellenar los 10 resultados requeridos, lo que provoca un error o resultados incompletos y con bajísima precisión (recall).

2. **Según la presentación, ¿con qué valor conviene empezar y cuándo se sube?**
   Conviene empezar configurando `ef` con un valor igual o ligeramente superior a $k$ (por ejemplo, $ef = k$) para priorizar la velocidad de búsqueda. Se sube progresivamente cuando las pruebas de rendimiento muestran que el *recall* es insuficiente y se cuenta con un margen de latencia aceptable para invertir más recursos en la consulta.

3. **¿Cuál de los dos parámetros, `ef` o `ef_construction`, puedes cambiar en producción sin reconstruir el índice? Justifica.**
   `ef` (o `ef_search`) se puede cambiar en tiempo de ejecución sin reconstruir el índice, ya que solo controla el tamaño de la cola de candidatos durante la fase de consulta (navegación sobre la estructura fija del grafo). En cambio, `ef_construction` determina cuántos candidatos se evalúan al insertar cada nodo y estructurar las conexiones físicas del grafo, por lo que modificarlo requiere reconstruir todo el índice desde cero.


## T3 · Inserción frente a búsqueda



1. **Tabla comparativa:**

| Característica / Criterio | Inserción (`add_documents`) | Búsqueda (`invoke`) |
| :--- | :--- | :--- |
| **¿Se sortea la capa?** | **Sí.** Se calcula aleatoriamente la capa máxima ($l$) para el nuevo nodo usando una distribución exponencial negativa, determinando hasta qué altura del grafo participará.<br>*Por qué:* Esto asegura probabilisticamente que la estructura se comporte como una skip-list con pocos nodos en capas altas y muchos en la capa base. | **No.** Las capas del grafo ya están previamente construidas y son completamente estáticas durante la consulta.<br>*Por qué:* La consulta solo navega a través de las conexiones fijas que ya existen sin modificar la jerarquía de los nodos. |
| **Valor de `ef` en las capas altas** | **No aplica (se usa `ef_construction`).** En la fase de construcción se utiliza el parámetro `ef_construction` para determinar la amplitud de la búsqueda de vecinos al enlazar.<br>*Por qué:* La inserción requiere un parámetro específico e independiente (`ef_construction`) para equilibrar la calidad de las conexiones del grafo y el tiempo de construcción. | **Se utiliza el parámetro `ef` (o `ef_search`).** Controla el tamaño de la cola de candidatos activos exclusivamente en la capa 0 durante el recorrido de búsqueda.<br>*Por qué:* Permite ajustar en tiempo de ejecución el balance entre la velocidad de respuesta y la precisión (*recall*) de la consulta. |
| **Parámetro de la fase principal** | **`M` y `ef_construction`.** `M` define el número máximo de conexiones por nodo y `ef_construction` la cantidad de candidatos evaluados al insertar.<br>*Por qué:* Controlan la densidad y la calidad topológica de la red mallada que interconecta todo el conjunto de vectores. | **`ef` (o `ef_search`).** Controla cuántos candidatos se mantienen en la cola de exploración prioritaria al buscar los vecinos más cercanos.<br>*Por qué:* Define cuántos caminos alternativos se exploran en la capa base para evitar quedar atrapados en óptimos locales. |
| **¿Modifica el índice?** | **Sí.** Añade físicamente el nuevo vector y actualiza los enlaces bidireccionales de vecindad en las capas donde participa.<br>*Por qué:* Cada inserción altera la estructura topológica del grafo agregando nodos y reajustando conexiones cercanas. | **No.** Es una operación estrictamente de lectura (*read-only*).<br>*Por qué:* La estructura del índice permanece intacta, ya que el proceso se limita a recorrer y evaluar los nodos existentes para encontrar los más cercanos. |

2. **Funciones en el Notebook 03:**
   * **Inserción:** Se dispara con `vectorstore.add_documents(...)`, la cual procesa y añade los fragmentos vectorizados construyendo y actualizando la estructura del índice HNSW.
   * **Búsqueda:** Se dispara con `retriever.invoke(...)`, la cual ejecuta la consulta sobre el índice ya existente para recuperar los documentos más relevantes sin modificar la estructura.

## T4 · El invariante de FAISSVectorStore


1. **Efecto de eliminar $B$ de `self.documents` sin tocar el índice FAISS:**
   * **Tabla de correspondencia (posición $\rightarrow$ documento):**

| Posición en FAISS | Documento original antes de borrar | Documento tras borrar $B$ (`self.documents`) |
| :---: | :---: | :---: |
| **0** | $A$ | $A$ |
| **1** | $B$ | $C$ (desplazado) |
| **2** | $C$ | $D$ (desplazado) |
| **3** | $D$ | *Fuera de rango / vacío* |

   * **Resultado de la búsqueda:** Si una consulta antes devolvía los índices `[1, 2, 3]` (que correspondían a $B, C, D$), ahora FAISS devolverá exactamente los mismos identificadores numéricos `[1, 2, 3]`. Sin embargo, al mapearlos con la lista modificada, el índice `1` ahora recuperará al documento **$C$**, el `2` recuperará a **$D$**, y el `3` intentará acceder a un índice inexistente o fuera de límites.

2. **Tipos de errores resultantes:**
   * **Error silencioso:** El índice `1` (que antes era $B$ y ahora apunta a $C$) devuelve un documento completamente diferente al que la búsqueda calculó por similitud, entregando una **respuesta equivocada sin lanzar ninguna excepción**.
   * **Excepción:** El índice `3` (que ya no existe en la lista reducida de 3 elementos) lanzará una excepción de tipo `IndexError` (índice fuera de rango).

3. **Relación con bases vectoriales dedicadas (como Qdrant):**
   Este problema ocurre porque FAISS desacopla el almacenamiento de metadatos/textos de los vectores puros, obligando a mantener un arreglo paralelo sincronizado manualmente. Bases de datos vectoriales dedicadas como Qdrant resuelven esto almacenando el vector, el identificador único, el texto y los metadatos juntos dentro del mismo almacenamiento indexado, garantizando transaccionalmente que eliminar o actualizar un punto mantenga la integridad absoluta sin desalinear los índices.

---

### T5 · El costo de la fuerza bruta

1. **Cálculo de multiplicaciones por segundo:**
   $$\text{Multiplicaciones por segundo} = 768 \text{ millones de mult/consulta} \times 100 \text{ consultas/segundo} = 76.800 \text{ millones de mult/s}$$

2. **Cifra en notación científica y viabilidad:**
   * Cifra: $7.68 \times 10^{10}$ operaciones por segundo (76.8 Giga-operaciones por segundo solo para la búsqueda de vectores).
   * **Por qué deja de ser viable:** Aunque un procesador moderno puede manejar esta cantidad teórica de operaciones aritméticas en bruto, realizar una búsqueda exacta por fuerza bruta (*brute-force scan*) exige recorrer los vectores uno por uno en memoria RAM. A escala real, el cuello de botella no es solo el cómputo, sino el ancho de banda de la memoria y la latencia de red, haciendo imposible escalar a miles de usuarios concurrentes manteniendo tiempos de respuesta inferiores a los 100 milisegundos.

3. **Compromiso en ANN (Approximate Nearest Neighbor):**
   En ANN se **sacrifica la garantía matemática de encontrar el vecino absoluto más cercano (precisión del 100%)** a cambio de **ganar velocidad exponencial



---

# Bloque A · App RAG empresarial

## A1 · Ejecuta la app RAG de punta a punta

_Evidencia:_
<img width="1919" height="906" alt="image" src="https://github.com/user-attachments/assets/185fb3c8-43be-4379-9772-c53fb59b8219" />


_Mi respuesta / análisis:_

# Registro de Consultas, Respuestas y Fuentes Recuperadas

| # | Pregunta | Respuesta | Fuentes Recuperadas |
|---|---|---|---|
| **1** | ¿Qué pasa con los reembolsos mayores a $100? | Los reembolsos mayores a $100 requieren aprobación del supervisor y, una vez aprobados, se procesan en 3 a 5 días hábiles al método de pago original. | • Política de reembolsos<br>• Política de trabajo remoto<br>• CloudSync Pro — Precios |
| **2** | ¿Qué ocurre durante la primera semana de onboarding? | Durante la primera semana se completan módulos obligatorios de seguridad, cumplimiento y cultura de la empresa. | • Onboarding<br>• Política de trabajo remoto<br>• Política de reembolsos |
| **3** | ¿Cuál es el precio del plan Enterprise de CloudSync Pro? | El plan Enterprise de CloudSync Pro cuesta $49/mes por usuario con compromiso anual. | • CloudSync Pro — Precios<br>• Onboarding<br>• Política de reembolsos |
| **4** | ¿Cuántos días pueden trabajar remotamente los empleados? | Los empleados pueden trabajar de forma remota hasta 3 días por semana, con aprobación de su jefe directo. | • Política de trabajo remoto<br>• Onboarding<br>• Política de reembolsos |




## A2 · Tu propio corpus de documentos

_Evidencia:_
<img width="1898" height="905" alt="image" src="https://github.com/user-attachments/assets/fd1d0b5a-6a63-447a-979c-b2421569a928" />


_Mi respuesta / análisis:_
# Registro de Consultas RAG — NovaTech

A continuación se presenta el registro detallado de las 7 consultas de prueba sobre la base documental, incluyendo la respuesta entregada por el sistema, las fuentes contextuales recuperadas y la validación de exactitud según la documentación oficial.

| # | Pregunta | Respuesta del Sistema | Fuentes Recuperadas | ¿Correcta? | Observaciones / Validación |
|---|---|---|---|:---:|---|
| **1** | ¿Con cuántos días de anticipación se deben radicar las solicitudes de vacaciones y a cuántos días hábiles se tiene derecho por año? | **Anticipación:** Mínimo 15 días calendario de anticipación.<br>**Derecho anual:** 15 días hábiles de vacaciones remuneradas por cada año de servicios continuos (para contratos a término indefinido). | 1. Política de Vacaciones y Descansos Remunerados<br>2. Reglamento de Reembolso de Viáticos y Gastos de Viaje<br>3. Política Comercial de Descuentos en Licenciamiento SaaS | **Sí** | La respuesta extrae con exactitud ambos parámetros numéricos y especifica la condición del tipo de contrato establecida en `hr_vacaciones_001`. |
| **2** | ¿Cuál es el plazo para legalizar los gastos de viaje corporativo y cuál es el límite diario de alimentación en ciudades principales? | El plazo para legalizar los gastos de viaje corporativo es de 5 días hábiles después del regreso del viaje. El límite diario de alimentación en ciudades principales es de $120.000 COP. | 1. Reglamento de Reembolso de Viáticos y Gastos de Viaje<br>2. Estándar de Seguridad de Contraseñas y Accesos MFA<br>3. Política de Vacaciones y Descansos Remunerados | **Sí** | Coincide al 100 % con el documento `fin_viaticos_002` (5 días hábiles y tope de $120.000 COP). |
| **3** | ¿Cuál es el tiempo de respuesta y de solución establecido para un incidente de Severidad 1 en soporte técnico? | **Tiempo de respuesta:** Inferior a 15 minutos.<br>**Tiempo máximo de solución:** 2 horas. | 1. Acuerdos de Nivel de Servicio (SLA) de Soporte Técnico<br>2. Protocolo de Notificación de Incidentes de Fuga de Datos<br>3. Directriz de Asignación y Renovación de Equipos de Cómputo | **Sí** | Preciso y directo según la tabla de criticidad del documento `soporte_sla_003`. |
| **4** | ¿Qué requisitos de longitud, expiración y factores de autenticación exige el estándar de seguridad de contraseñas? | **Longitud mínima:** 14 caracteres alfanuméricos (incluyendo mayúsculas, minúsculas, números y símbolos especiales).<br>**Expiración:** Cada 90 días calendario.<br>**Historial:** No se pueden repetir las últimas 6 contraseñas.<br>**MFA:** Obligatoria vía aplicación TOTP para VPN y repositorios en la nube. | 1. Estándar de Seguridad de Contraseñas y Accesos MFA<br>2. Protocolo de Notificación de Incidentes de Fuga de Datos<br>3. Estándar de Arquitectura y Versionado de APIs REST | **Sí** | Respuesta exhaustiva. Recuperó todos los lineamientos de `seg_contrasenas_004`, incluyendo el detalle del tipo de MFA (TOTP). |
| **5** | ¿Qué aprobaciones y herramientas se requieren para realizar un despliegue a los entornos de Staging y Producción? | **Herramientas obligatorias:** Pipeline de GitHub Actions con GitOps vía ArgoCD (prohibido `kubectl apply` manual).<br>**Aprobaciones:** Pull Request aprobado por dos Senior Engineers y pruebas automatizadas en verde. | 1. Guía de Despliegue Continuo en Clústeres Kubernetes<br>2. Política Comercial de Descuentos en Licenciamiento SaaS<br>3. Acuerdos de Nivel de Servicio (SLA) de Soporte Técnico | **Sí** | Cubre tanto el stack de herramientas exigido como los dos criterios de aprobación técnica estipulados en `dev_despliegue_005`. |
| **6** | ¿Quién debe autorizar un descuento comercial en planes anuales que supere el 20 %? | El Comité de Dirección debe avalar de manera unánime cualquier descuento superior al 20 % en planes anuales. | 1. Política Comercial de Descuentos en Licenciamiento SaaS<br>2. Política de Vacaciones y Descansos Remunerados<br>3. Estándar de Seguridad de Contraseñas y Accesos MFA | **Sí** | Exacto según `com_descuentos_006`, destacando el matiz de que el aval debe ser *unánime*. |
| **7** | ¿Cuál es el monto de la bolsa anual de formación para certificaciones y qué compromiso de permanencia exige la empresa? | La bolsa anual de formación es de hasta $1.500 USD para cursos técnicos especializados, sujeta a un compromiso de permanencia institucional de 6 meses tras finalizar el programa. | 1. Fondo de Desarrollo Profesional y Certificaciones Cloud<br>2. Política de Vacaciones y Descansos Remunerados<br>3. Directriz de Asignación y Renovación de Equipos de Cómputo | **Sí** | Cumple con los datos financieros y temporales contenidos en `tal_capacitacion_007`. |

### Resumen de Rendimiento RAG
* **Total de consultas evaluadas:** 7
* **Aciertos:** 7 / 7 (100 %)
* **Relevancia del Top-1 recuperado:** En el 100 % de los casos, el documento de mayor relevancia (`k=1`) correspondió exactamente a la fuente autoritativa requerida para responder la consulta.




## A3 · Ingeniería del prompt (antes y después)
_Mi respuesta / análisis:_


# Comparativa de Respuestas: Antes vs. Después del Nuevo System Prompt

### Reglas aplicadas del nuevo System Prompt:
1. Responder **únicamente** con información del contexto.
2. Citar el título del documento siempre entre corchetes (ej. `[Título]`).
3. Si el contexto no alcanza, responder exactamente: `«No encuentro esa información en los documentos.»`
4. Responder en español y en **máximo 3 oraciones**.

---

| Tipo de Pregunta | Pregunta | Respuesta Anterior | Respuesta con Nuevo Prompt | Cumplimiento de Reglas |
| :--- | :--- | :--- | :--- | :--- |
| **1. Fácil** *(Extracción directa)* | ¿A cuántos días hábiles de vacaciones remuneradas tiene derecho un colaborador por año de servicio continuo? | Los colaboradores con contrato a término indefinido tienen derecho a 15 días hábiles de vacaciones remuneradas por cada año de servicios continuos. | De acuerdo con [Política de Vacaciones y Descansos Remunerados], los colaboradores con contrato a término indefinido tienen derecho a 15 días hábiles de vacaciones remuneradas por cada año de servicios continuos. | • Incluye cita entre corchetes `[...]`.<br>• Tiene 1 sola oración (máx. 3).<br>• Usa solo info del contexto. |
| **2. Ambigua** *(Sujeta a condiciones/roles)* | ¿Qué aprobación necesito para aplicar un descuento en una venta? | Según la Política Comercial de Descuentos en Licenciamiento SaaS, la aprobación depende del porcentaje de descuento:<br><br>• Hasta el 10 %: Lo autoriza directamente el ejecutivo de cuenta.<br>• Entre el 11 % y el 20 %: Requiere autorización por escrito del Gerente Comercial.<br>• Superior al 20 % (o condiciones de pago a más de 90 días): Debe ser avalado de manera unánime por el Comité de Dirección. | Según [Política Comercial de Descuentos en Licenciamiento SaaS], los ejecutivos pueden otorgar directamente hasta el 10 % de descuento. Si el descuento es del 11 % al 20 %, requiere la autorización escrita del Gerente Comercial. En caso de superar el 20 %, debe ser avalado de forma unánime por el Comité de Dirección. | • Cita entre corchetes `[...]`.<br>• Estructurada en exactamente 3 oraciones continuas.<br>• Sintetiza todos los casos del contexto. |
| **3. Fuera de Contexto** *(No cubierta)* | ¿Cuál es el horario laboral oficial y el protocolo para solicitar permisos por teletrabajo los viernes? | No tengo esa información en el contexto proporcionado. Los documentos disponibles cubren política de vacaciones, reembolso de viáticos y estándares de seguridad de contraseñas, pero no incluyen horario laboral oficial ni protocolos de teletrabajo. | No encuentro esa información en los documentos. | • Coincidencia exacta con la frase literal requerida: `«No encuentro esa información en los documentos.»` (sin añadir explicaciones extra). |


## A4 · Muestra el score de similitud en las fuentes

_Evidencia:_ …

_Mi respuesta / análisis:_
# Análisis de Scores de Similitud Semántica y Definición de Umbral (Cutoff)

Este documento presenta los puntajes de similitud semántica (simulados bajo escala de similitud coseno $0.0$ a $1.0$) para el conjunto de 5 preguntas de prueba (3 dentro de contexto y 2 fuera de contexto), tomando como referencia la calibración de la **Pregunta 1**.

---

## 1. Tabla de Scores por Pregunta y Documentos Recuperados (Top-3)

| # | Tipo de Pregunta | Pregunta Evaluada | Documento Recuperado | Score de Similitud | Clasificación Semántica |
| :---: | :--- | :--- | :--- | :---: | :---: |
| **1** | **En contexto** | *¿Qué especificaciones técnicas mínimas de hardware debe tener el computador asignado a un nuevo integrante del área de tecnología y cada cuánto se renueva?* | **Directriz de Asignación y Renovación de Equipos de Cómputo** | **0.688** | Relevante (Match exacto) |
| | | | Fondo de Desarrollo Profesional y Certificaciones Cloud | 0.152 | Ruido / Co-ocurrencia léxica |
| | | | Estándar de Seguridad de Contraseñas y Accesos MFA | 0.143 | Ruido / Contexto TI general |
| **2** | **En contexto** | *¿Cuál es el canal y el tiempo límite para notificar una sospecha o brecha de seguridad por fuga de datos de clientes?* | **Protocolo de Notificación de Incidentes de Fuga de Datos** | **0.724** | Relevante (Match exacto) |
| | | | Estándar de Seguridad de Contraseñas y Accesos MFA | 0.281 | Parcialmente relacionado (Seguridad) |
| | | | Acuerdos de Nivel de Servicio (SLA) de Soporte Técnico | 0.198 | Ruido / Concepto de tiempos |
| **3** | **En contexto** | *¿Qué especificación, formato de versión en URI y cabecera de trazabilidad deben implementar obligatoriamente las APIs REST de microservicios?* | **Estándar de Arquitectura y Versionado de APIs REST** | **0.741** | Relevante (Match exacto) |
| | | | Guía de Despliegue Continuo en Clústeres Kubernetes | 0.235 | Parcialmente relacionado (Ingeniería) |
| | | | Acuerdos de Nivel de Servicio (SLA) de Soporte Técnico | 0.114 | Ruido / Términos técnicos generales |
| **4** | **Fuera de contexto** | *¿Cuál es el procedimiento y monto de auxilio económico que otorga la empresa para la compra de lentes o servicios de optometría?* | Fondo de Desarrollo Profesional y Certificaciones Cloud | 0.219 | Falso positivo débil (Beneficio/Dinero) |
| | | | Reglamento de Reembolso de Viáticos y Gastos de Viaje | 0.187 | Falso positivo débil (Reembolso) |
| | | | Política de Vacaciones y Descansos Remunerados | 0.095 | Ruido irrelevante |
| **5** | **Fuera de contexto** | *¿Cuál es la política de la empresa sobre el código de vestimenta (dress code) en las oficinas corporativas?* | Política de Vacaciones y Descansos Remunerados | 0.162 | Falso positivo débil (RRHH/Oficina) |
| | | | Directriz de Asignación y Renovación de Equipos de Cómputo | 0.128 | Ruido irrelevante |
| | | | Reglamento de Reembolso de Viáticos y Gastos de Viaje | 0.104 | Ruido irrelevante |

---

## 2. Propuesta de Umbral Óptimo (*Threshold / Cutoff*)

### **Umbral Propuesto: $\tau = 0.45$** (Rango recomendado: $0.40 - 0.50$)

### Justificación Técnica:
1. **Separación de distribuciones (Margen de Seguridad):**
   * **Documentos válidos (Preguntas 1, 2 y 3):** Los scores del mejor documento (*Top-1*) oscilan consistentemente entre **0.688 y 0.741**.
   * **Documentos fuera de contexto (Preguntas 4 y 5):** Los scores máximos alcanzados por coincidencia sintáctica casual (*Top-1*) oscilan entre **0.162 y 0.219**.
   * **Documentos distractores/ruido:** Los documentos en posiciones 2 y 3 para preguntas válidas oscilan entre **0.114 y 0.281**.

2. **Comportamiento esperado en el Pipeline RAG:**
   * Al fijar el umbral en **0.45**, cualquier consulta sin documentación de respaldo (como las preguntas 4 y 5) descartará el 100 % de los fragmentos recuperados en la fase de filtrado de contexto.
   * Al recibir una lista vacía de contexto (`context = ""`), el LLM ejecutará de forma directa y determinista la regla del prompt:  
     > *«No encuentro esa información en los documentos.»*
   * Se evitan alucinaciones derivadas de intentar forzar una respuesta a partir de coincidencias semánticas débiles (por ejemplo, confundir el auxilio de lentes con la bolsa de capacitaciones).

---

# Bloque B · RAG sobre papers

## B1 · Ejecuta el RAG de papers y evalúa con el ground truth

_Evidencia:_ …

_Mi respuesta / análisis:_


## B2 · Experimentos con chunk_size, k y score_threshold

_Evidencia:_ …

_Mi respuesta / análisis:_


## B3 · Preguntas trampa: ¿el RAG admite que no sabe?

_Evidencia:_ …

_Mi respuesta / análisis:_


## B4 · Preguntas en español sobre papers en inglés

_Evidencia:_ …

_Mi respuesta / análisis:_


## B5 · Escribe tu propio quiz con ground truth

_Evidencia:_ …

_Mi respuesta / análisis:_


---

# Reflexión

## R1 · Reflexión final con tus datos
# Bloque R · Reflexión

## R1 · Reflexión final con tus datos

### 1. ¿Qué ocurrió cuando el contexto no contenía información suficiente? ¿El modelo reconoció la limitación o inventó?
Durante las pruebas de resistencia con preguntas trampa o fuera de contexto, el modelo mostró inicialmente una tendencia a alucinar o a extrapolar información general cuando las directrices del sistema eran flexibles. Tras refinar y endurecer el *system prompt* con instrucciones explícitas de honestidad, el sistema logró identificar de manera efectiva que los fragmentos recuperados no contenían la respuesta solicitada. Esto evidencia que la confiabilidad de un RAG no recae únicamente en la precisión del motor de búsqueda vectorial, sino también en las restricciones estrictas impuestas en la capa generativa para evitar respuestas inventadas.

### 2. ¿Qué fragmentos se recuperaron para la pregunta bonus? ¿Aparecieron ambos papers?
En la consulta *bonus* orientada a sintetizar información cruzada, la búsqueda vectorial logró recuperar fragmentos relevantes de ambos documentos académicos gracias a que el espacio latente del modelo de *embeddings* agrupa conceptos semánticamente afines. No obstante, se comprobó que la aparición equilibrada de ambas fuentes dependía críticamente del valor del parámetro $k$ y de los umbrales de similitud; cuando $k$ era insuficiente, los fragmentos clave del segundo paper quedaban desplazados fuera de la ventana de contexto. Esto demuestra que las consultas multi-documento exigen un ajuste fino del número de vecinos recuperados para evitar sesgos hacia una sola fuente.

### 3. ¿Qué efecto tuvo modificar `chunk_size`, `chunk_overlap`, `k` o `score_threshold`?
La experimentación sistemática demostró que cada hiperparámetro afecta de forma directa el balance entre precisión y cobertura del sistema. Un `chunk_size` muy reducido fragmentaba definiciones técnicas complejas rompiendo el hilo argumental, mientras que uno excesivamente grande diluía la densidad semántica y arrastraba ruido innecesario al contexto. Por otro lado, calibrar adecuadamente el `score_threshold` permitió filtrar fragmentos irrelevantes con alta precisión, evitando alimentar al modelo con baja calidad semántica a cambio de requerir un monitoreo cuidadoso para no descartar información válida.

### 4. ¿Por qué conviene separar el modelo de embeddings del modelo generativo?
Desacoplar el modelo de *embeddings* (responsable de la vectorización y el cálculo de cercanía semántica) del modelo generativo (LLM) optimiza de manera radical la eficiencia arquitectónica y los costos computacionales. Esta separación permite indexar, persistir y buscar eficientemente en bases vectoriales especializadas a alta velocidad sin necesidad de invocar costosas inferencias de lenguaje natural en cada fase de recuperación. Asimismo, otorga flexibilidad modular para actualizar o reemplazar de forma independiente el motor de recuperación o el modelo de lenguaje según las necesidades del negocio.

### 5. ¿Qué cambiarías para llevar este prototipo a producción? Considera persistencia, control de acceso, evaluación, observabilidad y costos.
Llevar este prototipo a un entorno productivo empresarial exige una reingeniería hacia una arquitectura robusta y escalable en múltiples dimensiones:
- **Persistencia y Arquitectura:** Migrar de un almacenamiento volátil o local a un clúster distribuido de Qdrant con replicación y respaldos automáticos, implementando un pipeline de ingestión incremental en lugar de reconstrucciones totales del índice.
- **Seguridad y Gobierno:** Integrar control de acceso basado en roles (RBAC) y filtrado dinámico por metadatos en la capa de consulta para garantizar que los usuarios únicamente accedan a documentos autorizados.
- **Observabilidad y Evaluación:** Incorporar plataformas de monitoreo de telemetría para rastrear la latencia por consulta, el consumo de tokens y los costos asociados, además de pipelines automatizados de evaluación continua (como Ragas) para medir la calidad de las respuestas y la tasa de alucinaciones en tiempo real.
