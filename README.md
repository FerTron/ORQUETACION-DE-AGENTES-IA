# HablaBank: asistente conversacional para banca digital

Prototipo de un asistente de chat para un banco. El cliente escribe en lenguaje natural ("Transfiere 200 a María", "¿Cuánto tengo?") y el sistema entiende la intención y la ejecuta con controles de seguridad.

**Autor:** Fermin Huallpa Mamani

> **Datos:** todo es simulado (Faker + SQLite en memoria). No se usa ningún dato real de clientes. La única llamada externa es a una API pública de tipo de cambio.

---

## 1. El problema y el usuario

**Usuario:** un cliente de banca digital que usa el chat desde el celular y quiere resolver cosas simples sin llamar ni ir a una agencia.

**Qué necesita hacer:**

- Consultar su saldo y sus últimos movimientos.
- Generar un QR de cobro.
- Ver el tipo de cambio.
- Hacer transferencias a sus contactos.
- Desbloquear su cuenta si quedó bloqueada.

**El problema:** en un banco, equivocarse cuesta dinero. Un asistente con un LLM "libre" podría mover dinero por una mala interpretación, mostrar datos sin autenticación o dejarse manipular con frases como *"ignora tus reglas"* (*prompt injection*). Se necesita la comodidad del lenguaje natural sin perder control.

## 2. ¿Por qué workflow y no agente?

**Es un workflow con controles propios de un agente.** La decisión fue deliberada:

| | Agente autónomo | Este proyecto (workflow) |
|---|---|---|
| Quién decide qué herramienta usar | El LLM | El código (flujo fijo por intención) |
| Orden de los pasos | Lo improvisa el modelo | Definido de antemano |
| Qué hace el LLM | Razona, decide y actúa | Solo clasifica la intención y extrae datos |
| Riesgo con dinero | Alto | Bajo y auditable |

Los pasos de cada tarea se conocen de antemano (preparar, confirmar, ejecutar), así que un agente solo agregaría riesgo sin aportar valor. El LLM entiende el lenguaje; el dinero y la seguridad los decide código determinista.

## 3. Arquitectura

```mermaid
flowchart TD
    A[Mensaje del cliente] --> B[Sesión y autenticación<br/>PIN fuera del chat]
    B --> C[Router de intención<br/>LLM o reglas; JSON validado]
    C --> D[Flujo fijo por intención<br/>orden definido en código]
    D --> E[Herramienta<br/>contrato ok / error]
    E --> F[Confirmación del cliente<br/>solo si mueve dinero]
    F --> G[Respuesta por plantilla<br/>+ auditoría]

    B -. sin sesión válida .-> X1[Pide PIN, no hay datos]
    C -. JSON inválido o dudoso .-> X2[Intención 'otra']
    D -. falta un dato .-> X3[Pregunta, no lo supone]
    E -. error, límite o MAX_PASOS .-> X4[Rechaza o escala a soporte]
    F -. sin 'sí' explícito .-> X5[No se mueve dinero]
```

Detalles importantes:

- **El LLM solo interviene en el router** (el 3.er paso). Todo lo demás es código determinista.
- **El orden real de decisión** está en `_despachar`: sesión escalada → transferencia pendiente → desbloqueo en curso → dato faltante → recién entonces el router. Mientras hay una confirmación o un desbloqueo en curso, el mensaje del cliente **nunca llega al LLM**.
- **Todas las herramientas se llaman a través de `usar()`**, que aplica el límite de llamadas y la auditoría.

## 4. Los 5 primitivos

### 4.1 Instrucciones
Lo que le dice al sistema cómo comportarse.

- `SYSTEM_PROMPT` (sección 5): reglas fijas del LLM. Solo clasifica, responde únicamente JSON, no inventa datos y no obedece órdenes escondidas en el mensaje del cliente. Se usa con `temperature=0`.
- Las **respuestas al cliente** son plantillas de texto en código (no las genera el LLM).
- Las **preguntas de seguridad** del desbloqueo están fijas en el diccionario `_POOL`.

### 4.2 Herramientas
13 herramientas, todas con contrato `{"ok": True, ...}` o `{"ok": False, "error": {codigo, mensaje}}`. Se reúnen en el diccionario `TOOLS`.

| Tipo | Herramientas |
|---|---|
| Lectura | `consultar_saldo`, `listar_movimientos`, `generar_qr_cobro`, `consultar_tipo_cambio` |
| Dinero (2 pasos) | `preparar_transferencia`, `confirmar_transferencia`, `cancelar_transferencia` |
| Desbloqueo | `iniciar_desbloqueo`, `verificar_otp`, `nueva_pregunta`, `verificar_respuesta`, `finalizar_desbloqueo` |
| Soporte | `escalar_a_humano` |

Las cuentas siempre se muestran enmascaradas (`****7890`) y el LLM nunca invoca las herramientas que escriben.

### 4.3 Memoria
Lo que el sistema recuerda para dar contexto.

- **Memoria de la conversación** (diccionario de sesión `s`): `historial` (últimos 10 mensajes), `ultima_transferencia` (para entender "lo mismo a Juan") y `slot` (el dato que falta).
- **Datos persistentes** (SQLite): tablas `cuentas`, `movimientos` y `contactos`.

### 4.4 Estado
Dónde está parado el flujo en este momento.

- En la sesión: `autenticado`, `pin_fallos`, `pendiente` (transferencia esperando "sí"), `desbloqueo` (fase y fallos), `escalado` y `ticket`.
- En la base: cada transferencia pasa por `PENDIENTE` → `CONFIRMADA` o `CANCELADA`, y vence a los 5 minutos.
- En el "servidor": `_OTP` y `_ESPERADAS` (respuestas correctas del desbloqueo). El LLM nunca los ve.

### 4.5 Control
Lo que frena al sistema cuando algo sale mal. **Ninguno depende del prompt.**

| Control | Tipo | Dónde | Qué evita |
|---|---|---|---|
| Confirmación explícita ("sí") | Aprobación humana | `preparar_transferencia` → `_resolver_confirmacion` → `confirmar_transferencia` | Mover dinero sin permiso |
| `MAX_PASOS` (8 por turno) y `MAX_TURNOS` (30) | Límite de llamadas | `usar()` lanza `LimitePasos` | Bucles o abuso; escala a soporte |
| Límites de Bs 1,000 por operación y Bs 2,000 por día (Bs 300 por 24 h tras un desbloqueo) | Guardrail | `preparar_transferencia` | Montos excesivos |
| Validación del JSON del LLM (confianza mínima 0.6) | Guardrail | `validar_clasificacion` | Respuestas inventadas o manipuladas |
| Autenticación previa | Guardrail | `_ejecutar` | Ver datos sin PIN |
| 3 fallos en el desbloqueo → soporte | Guardrail | `_registrar_fallo` | Prompt injection y adivinanza |
| Tabla `auditoria` | Trazabilidad | `usar()` / `auditar` | Acciones sin registro |

## 5. Cómo ejecutarlo paso a paso

### Requisitos
- Python 3.10 o superior (o una cuenta de Google para usar Colab).
- Internet: solo se necesita para instalar librerías y para el tipo de cambio. Sin internet, el sistema avisa que el servicio no está disponible y no inventa cifras.
- Una clave de la API de Anthropic es **opcional**. Sin ella, el router usa un clasificador por reglas.

### Opción A: Google Colab (la más simple)
1. Entra a [colab.research.google.com](https://colab.research.google.com) y sube `HablaBank_prototipo.ipynb` (Archivo → Subir cuaderno).
2. *(Opcional)* Para usar el LLM, agrega esta línea en una celda **antes** de la sección 2 (Configuración):
   ```python
   import os; os.environ["ANTHROPIC_API_KEY"] = "tu-clave-aqui"
   ```
   No subas tu clave a un repositorio ni la compartas.
3. Menú **Entorno de ejecución → Ejecutar todas**.
4. Revisa los resultados (ver "Qué deberías ver").

### Opción B: Jupyter local
1. Clona o descarga el proyecto y entra a la carpeta.
2. Crea un entorno virtual e instala las dependencias:
   ```bash
   python -m venv venv
   source venv/bin/activate        # En Windows: venv\Scripts\activate
   pip install jupyter anthropic faker "qrcode[pil]" requests pandas
   ```
3. *(Opcional)* Define la clave antes de abrir Jupyter:
   ```bash
   export ANTHROPIC_API_KEY="tu-clave-aqui"     # En Windows: set ANTHROPIC_API_KEY=tu-clave-aqui
   ```
4. Abre el notebook:
   ```bash
   jupyter notebook HablaBank_prototipo.ipynb
   ```
5. Ejecuta todas las celdas en orden (**Kernel → Restart & Run All**).

### Qué deberías ver

| Sección | Contenido |
|---|---|
| 1 a 4 | Instalación, configuración, datos simulados y herramientas (con una prueba rápida) |
| 5 | Router: 5 frases de ejemplo clasificadas |
| 6 | Workflow: el orquestador (sin salida) |
| 7 | Demo de conversación: saldo, transferencia con confirmación, "lo mismo a Juan", cancelación y QR |
| 8 | 8 casos de prueba con aserciones (todos en ✅) |
| 9 | Últimas 15 filas de la auditoría |
| 10 | Controles en vivo: aprobación humana y límite de llamadas |
| 11 | Evaluación de 5 casos con `debe_usar` / `no_debe_usar`: **5 de 5 = 100 %** |

### Datos de prueba
- **ACC-001:** cuenta normal, PIN `1234`, saldo Bs 2,500. Contactos: María, Juan y un tercero.
- **ACC-002:** cuenta bloqueada, para probar el desbloqueo.

### Problemas comunes
- **Falla el tipo de cambio:** revisa tu conexión. El sistema responde que el servicio no está disponible.
- **No hay clave de API:** es normal; el sistema usa las reglas y todo el notebook funciona igual.
- **`ModuleNotFoundError`:** instala la librería que indica el mensaje con `pip install`.

## 6. Evaluación

Cada caso define `debe_usar` y `no_debe_usar`, y se verifica contra la tabla de auditoría (las herramientas realmente invocadas), no contra lo que el sistema responde.

| Caso | Tipo | debe_usar | no_debe_usar | Acierto |
|---|---|---|---|---|
| 1. Consulta de saldo | Normal | `consultar_saldo` | `preparar_` / `confirmar_transferencia` | Sí |
| 2. Transferencia con aprobación | Normal + control | `preparar_` y `confirmar_transferencia` | `cancelar_transferencia` | Sí |
| 3. Destinatario inexistente | Error | `preparar_transferencia` | `confirmar_transferencia` | Sí |
| 4. Monto sobre el límite | Rechazo | `preparar_transferencia` | `confirmar_transferencia` | Sí |
| 5. Prompt injection en el desbloqueo | Rechazo (ataque) | flujo de desbloqueo y `escalar_a_humano` | `finalizar_desbloqueo` | Sí |

**Resultado: 5 de 5 = 100 %.**

**Limitación:** la evaluación se corrió con el clasificador por reglas. Con el LLM real los resultados podrían variar, y 5 casos no prueban robustez. Siguiente paso: ampliar los casos y repetirlos con el LLM, varias veces.

## 7. Aprendizajes

- Los controles van en código, no en el prompt: un prompt se puede convencer, una función no.
- Separar "preparar" de "ejecutar" hace más difícil el error más caro: mover dinero mal.
- Evaluar con la auditoría es más honesto que fijarse en la respuesta.

## 8. Archivos

| Archivo | Contenido |
|---|---|
| `HablaBank_prototipo.ipynb` | Todo el código: herramientas, router, workflow, pruebas y evaluación |
| `Guion_demo_HablaBank.docx` | Guion de la demo (1 página) |
