# ADR-003 — Estrategia de Extracción de Texto

**Estado:** Enfoque recomendado documentado — implementación pendiente

---

## Decisión

**Enrutar por formato de entrada — no existe una única estrategia de extracción.**

- **PDF:** ruta rápida híbrida — PdfPig primero, Amazon Textract como respaldo para páginas escaneadas.
- **Formatos de texto nativo** (Markdown, DOCX, PPTX, XLSX, texto plano): parsear directamente. Sin OCR involucrado — el texto ya es dato estructurado, no píxeles.
- **Imágenes sueltas** (PNG, JPEG, TIFF): directo a Amazon Textract — no existe una ruta "digital" rápida para una imagen independiente.

Amazon Textract se reserva exclusivamente para contenido basado en imágenes. Nunca se usa para formatos donde el texto ya existe como dato estructurado — de hecho, no los soporta (ver [Amazon Textract](#amazon-textract) abajo).

## Contexto

El alcance de DocLens es soporte flexible para formatos de documentación estándar — Markdown, PDF, DOCX, y otros — no solo PDF. Antes de que pueda ejecutarse el análisis semántico vía Amazon Bedrock, debe extraerse el texto crudo del formato que haya subido el tenant.

De esto surgen tres problemas de extracción arquitectónicamente distintos:

1. **PDF** puede o no tener texto embebido (digital vs. escaneado) — este es el problema original que resuelve esta ADR.
2. **Contenedores de texto nativo** (`.md`, `.txt`, `.docx`, `.pptx`, `.xlsx`) ya contienen el texto como dato estructurado. No hay nada que "extraer" vía OCR o parsing de PDF — el archivo solo necesita leerse con el parser apropiado para su formato.
3. **Imágenes sueltas** (una página escaneada subida como `.png`/`.jpg` independiente en lugar de envuelta en un PDF) no tienen texto estructurado en absoluto — el OCR es la única opción.

---

## PdfPig

PdfPig es una librería .NET de código abierto que lee la estructura interna de un archivo PDF y extrae texto directamente — sin llamada de red, sin modelo de ML, sin costo por página.

Un PDF almacena su texto como una serie de flujos de contenido que describen dónde se ubica cada carácter en la página. PdfPig lee esos flujos y reconstruye el texto. Esto solo funciona cuando el PDF fue generado por software (exportación de Word, impresión de navegador, factura digital). Si el texto nunca se embebió — porque el documento fue escaneado — no hay nada que extraer.

**Fortalezas**

- Costo cero — sin precio por página, sin cuota de API.
- Síncrono — el resultado está disponible de inmediato en la misma invocación Lambda.
- Sin dependencia externa — funciona sin acceso a red; adecuado para Lambdas aisladas en VPC.
- Rápido — la extracción de un PDF típico de 10 páginas toma milisegundos de un solo dígito.
- Bien adaptado a documentos digitales: contratos exportados de Word, escritos generados por sistemas internos, facturas electrónicas.

**Debilidades**

- No maneja documentos escaneados — si el PDF contiene solo imágenes de página, PdfPig devuelve texto vacío.
- La reconstrucción de layout es limitada — tablas, layouts multi-columna y texto rotado pueden salir en el orden incorrecto.
- Sin soporte nativo para imágenes o archivos de Word.

---

## Amazon Textract

Amazon Textract es un servicio gestionado de AWS que usa machine learning para detectar y extraer texto de imágenes de documentos. Funciona sobre PDFs escaneados, fotografías de documentos, y cualquier otra entrada basada en imágenes.

Para documentos multi-página, Textract se ejecuta como un job asíncrono (`StartDocumentTextDetection` + `GetDocumentTextDetection`). Para entradas de una sola página, existe una llamada síncrona `DetectDocumentText`. `AnalyzeDocument` ofrece extracción estructurada de tablas y campos clave-valor de formularios.

**Fortalezas**

- Maneja documentos escaneados — funciona con fotografías, PDFs escaneados e imágenes de baja resolución.
- Extracción de datos estructurados — `AnalyzeDocument` devuelve tablas y pares clave-valor de formularios como objetos estructurados.
- Alta precisión en entradas degradadas — páginas torcidas, iluminación desigual, fuentes mixtas.
- Nativo de AWS — se integra directamente con S3; no requiere descargar el archivo a memoria de Lambda.

**Debilidades**

- Costo — se cobra por página; significativo a escala, especialmente para contratos multi-página o lotes masivos.
- Asíncrono para documentos multi-página — `StartDocumentTextDetection` añade latencia y requiere sondeo interno o callback SNS/SQS. Esto ya no es un problema bloqueante: la Lambda Processor (ver [ADR-011](011-document-processing-trigger.md)) se ejecuta fuera de la presión de timeout de API Gateway y puede absorber el sondeo del job asíncrono de Textract internamente.
- Latencia de arranque en frío — incluso las llamadas síncronas añaden cientos de milisegundos.
- Dependencia del proveedor — Textract es específico de AWS; migrar fuera requiere reemplazar la capa de OCR.
- Sobredimensionado para PDFs digitales — ejecutar OCR sobre un PDF con texto embebido añade costo y latencia sin ganancia de calidad. Textract siempre rasteriza la página y ejecuta visión por computadora sobre ella — nunca verifica si la fuente ya tiene una capa de texto embebida.
- **Sin soporte de formatos de texto nativo.** Los formatos de entrada de Textract se limitan a `PDF`, `PNG`, `JPEG` y `TIFF` — no acepta `.docx`, `.pptx`, `.xlsx`, `.md` ni `.txt` bajo ninguna circunstancia. Es un servicio de visión por computadora; necesita píxeles para ejecutar su modelo. Estos formatos deben parsearse directamente (ver [Extracción de Texto Nativo](#extraccion-de-texto-nativo-markdown-docx-pptx-xlsx-texto-plano) abajo).

---

## Extracción de Texto Nativo (Markdown, DOCX, PPTX, XLSX, Texto Plano)

Estos formatos ya almacenan el texto como dato estructurado — el paso de extracción es un parseo, no un problema de OCR. Dos sub-casos:

**Texto plano (`.md`, `.txt`)**

Decodificar los bytes del archivo como UTF-8. Sin librería, sin lógica de parsing — el contenido *es* el texto. La sintaxis de Markdown (`#`, `**`, `-`) se deja tal cual; es una señal barata y útil para el prompt de Bedrock (encabezados y listas aportan estructura) en lugar de ruido a eliminar.

**Contenedores OOXML (`.docx`, `.pptx`, `.xlsx`)**

Son archivos ZIP de partes XML (Office Open XML). Leerlos requiere un parser real — [`DocumentFormat.OpenXml`](https://www.nuget.org/packages/DocumentFormat.OpenXml) (paquete oficial de Microsoft para .NET/NuGet para OOXML, licencia MIT, sin dependencia de red) recorre el XML del documento y devuelve texto de párrafos/celdas/diapositivas directamente. Es el equivalente OOXML de lo que PdfPig hace para PDF — parsing estructural directo, sin renderizado, sin modelo de ML.

**Esta ruta nunca cae de vuelta a Textract.** Un `.docx` no tiene un modo de falla "escaneado" en uso normal — el texto siempre está presente como XML estructurado.

**Debilidades**

- El formato binario legado `.doc` (Word pre-2007) **no** está cubierto por `DocumentFormat.OpenXml` — esa librería solo lee OOXML (`.docx`/`.pptx`/`.xlsx`). El `.doc` binario necesita una herramienta distinta (p. ej. un paso de conversión vía LibreOffice headless, o una librería comercial) o debería excluirse del alcance de v1. Esto es un gap abierto, no un caso resuelto.
- El layout no se preserva de la forma en que lo ve un lector humano — layouts multi-columna, cuadros de texto y tablas embebidas en DOCX/PPTX requieren recorrer deliberadamente la estructura OOXML.

---

## Comparación

| Dimensión | PdfPig | Amazon Textract | Parser de Texto Nativo (`DocumentFormat.OpenXml` / lectura plana) |
|---|---|---|---|
| Aplica a | PDF digital | PDF escaneado, PNG, JPEG, TIFF | `.md`, `.txt`, `.docx`, `.pptx`, `.xlsx` |
| Costo | Gratis | Precio por página | Gratis |
| Latencia | Milisegundos (en proceso) | Cientos de ms (síncrono) / segundos (job asíncrono) | Milisegundos (en proceso) |
| Contenido escaneado/imagen | No | Sí | No aplica |
| Extracción estructurada (tablas, formularios) | Limitada | Sí (`AnalyzeDocument`) | Sí, vía estructura OOXML |
| Complejidad asíncrona | Ninguna | Requerida para multi-página | Ninguna |
| Dependencia de AWS | Ninguna | Sí | Ninguna |

---

## Enfoque recomendado: Enrutar por formato

| Paso | Acción |
|---|---|
| 1 | Detectar el formato de entrada por `Content-Type` / extensión de archivo en el momento de `POST /documents/prepare` |
| 2a | **PDF:** intentar extracción con PdfPig en proceso. Si el texto extraído está bajo el umbral (~50 caracteres) → tratar como basado en imagen → recurrir a Amazon Textract |
| 2b | **Texto nativo** (`.md`, `.txt`, `.docx`, `.pptx`, `.xlsx`): parsear directamente — decodificación UTF-8 plana, o `DocumentFormat.OpenXml` para OOXML |
| 2c | **Imagen suelta** (`.png`, `.jpg`, `.tiff`): enviar directamente a Amazon Textract |

Esto mantiene la mayoría del procesamiento rápido y gratuito, reserva Textract para los casos donde es la única opción, y mantiene el código de llamada desacoplado de *cómo* se leyó un formato dado.

**Heurística de umbral (solo ruta PDF):** un conteo simple de caracteres es un punto de partida confiable. Un PDF escaneado produce cero o casi cero caracteres desde PdfPig. Un PDF digital con incluso una página de contenido produce cientos. Un umbral entre 20 y 100 caracteres cubre esta distinción de forma confiable.

---

## Consecuencias

- **`IOcrService` necesita volverse consciente del formato.** La interfaz actual está pensada para un contrato único PDF-entrada/texto-salida. Debería evolucionar hacia algo como `IContentExtractionService` con enrutamiento por formato detectado.
- **Nueva dependencia NuGet:** `DocumentFormat.OpenXml` para parsing OOXML.
- **`POST /documents/prepare`** valida el `Content-Type` contra la lista de formatos soportados (ver [ADR-005](005-upload-strategy.md)) — esa restricción está parametrizada por el formato subido.

---

## Preguntas abiertas

- ¿Qué proporción de documentos subidos se espera que sean escaneados vs. PDF digital vs. texto nativo? Esto afecta tanto la proyección de costo de Textract como cuánto esfuerzo de ingeniería merece la ruta OOXML respecto a la ruta PDF.
- ¿Debería usarse `AnalyzeDocument` (tablas/formularios) en lugar de `DetectDocumentText` para mejorar la precisión de extracción de campos en documentos legales estructurados (contratos, formularios)?
- Para documentos multi-página, ¿debería usarse el flujo de job asíncrono de Textract (`StartDocumentTextDetection`) en lugar del síncrono `DetectDocumentText`? El pipeline de procesamiento asíncrono (ver [ADR-011](011-document-processing-trigger.md)) ya está en su lugar, así que esto es viable.
- ¿Está el `.doc` binario legado en el alcance de v1? Si es así, ¿qué lo maneja — conversión vía LibreOffice headless, una librería comercial, u otra cosa?
- ¿Debería detectarse y enrutarse a Textract una imagen escaneada embebida como todo el cuerpo de un `.docx`/`.pptx`, o queda explícitamente fuera de alcance?
