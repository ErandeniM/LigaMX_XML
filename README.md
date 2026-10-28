<p align="right">
<i>Erandeni Mendívil Morales</i><br>
<i>Desarrollo de Sistemas III</i><br>
<i>Universidad de Sonora · Octubre de 2026</i>
</p>

# Liga MX en XML: resultados y estadísticas validados con XML Schema

Este proyecto define un formato XML para representar los resultados y las estadísticas de los partidos de la Liga MX, junto con un XML Schema (XSD) que fija las reglas de cualquier documento en ese formato: qué elementos lleva, en qué orden, cuántas veces aparece cada uno y **qué tipo de dato admite**. El caso de prueba son los partidos del **domingo 27 de septiembre de 2026**, de la jornada 10 del Torneo Apertura 2026.

## Contenido del repositorio

```text
liga-mx-xml/
├── README.md                    ← documentación del diseño (este archivo)
├── xml/
│   ├── resultados.xml           ← partidos del domingo 27/09/2026 (válido)
│   └── resultados-invalido.xml  ← diez pruebas con errores (inválido a propósito)
└── xsd/
    └── resultados.xsd           ← XML Schema que valida el formato
```

## Fuente de los datos

[ESPN México: resultados de la Liga MX del 27 de septiembre de 2026](https://www.espn.com.mx/futbol/resultados/_/liga/mex.1/fecha/20260927), consultada el 6 de octubre de 2026. Las horas están en tiempo del centro de México (UTC−06:00).

## 1. Estructura del documento XML

ESPN organiza la información en tres niveles: una jornada agrupa varios partidos, y cada partido tiene dos equipos, un marcador y las estadísticas de cada lado. El documento sigue la misma jerarquía:

```text
liga                         @nombre @torneo @temporada
└── jornada (1 a 17)         @numero
    └── partido (1 a 9)      @id @estado
        ├── fecha
        ├── hora             @zona
        ├── estadio          @ciudad
        ├── equipoLocal      @abreviatura
        │   ├── nombre
        │   └── estadisticas (0 o 1)
        │       ├── posesion, tiros, tirosAPuerta, faltas, tirosDeEsquina
        │       ├── fuerasDeLugar (0 o 1), atajadas (0 o 1)
        │       └── tarjetas
        │           └── tarjeta (0 o más)   @color @minuto @agregado
        ├── equipoVisitante  @abreviatura   (mismo contenido que equipoLocal)
        └── marcador (0 o 1)
            ├── golesLocal, golesVisitante
            └── gol (0 o más)               @equipo @minuto @agregado @tipo
```

Decisiones de diseño:

- **El local y el visitante son elementos distintos** (`equipoLocal` y `equipoVisitante`), no dos `equipo` con un atributo. Así el schema exige exactamente uno de cada uno, en ese orden, y los dos comparten el mismo tipo.
- **Las estadísticas van dentro de cada equipo**, porque cada dato pertenece a un lado: Pumas hizo 15 tiros y San Luis 11. El mismo bloque sirve para ambos equipos.
- **Cada gol y cada tarjeta se registran por separado**, con el minuto y el jugador. El marcador guarda además los totales (`golesLocal` y `golesVisitante`), porque el detalle de los goles es opcional.
- **El minuto y el tiempo agregado van separados.** El 45+1 se escribe `minuto="45" agregado="1"`. Así los dos valores son enteros que el schema puede validar y los eventos se ordenan numéricamente.
- **Elemento o atributo.** Los datos del partido son elementos. Son atributos los valores que identifican o clasifican algo (`id`, `estado`, `color`, `equipo`, `tipo`) y los que precisan otro dato (la `ciudad` del estadio, la `zona` de la hora).

## 2. XML Schema (`xsd/resultados.xsd`)

### Organización

El schema declara **un solo elemento global, `liga`**, y define todo lo demás con **tipos con nombre**, un diseño conocido como *Venetian blind* (persiana veneciana):

- `liga` es el único elemento global, así que es el único que puede ser raíz. Un documento que empiece con `<jornada>` no es válido.
- Los tipos se reutilizan. `tipoEquipo` sirve para `equipoLocal` y para `equipoVisitante`. El grupo de atributos `momentoDelPartido` (`minuto` y `agregado`) sirve para `tarjeta` y para `gol`. En el DTD esas declaraciones se escribían dos veces.
- El schema no declara espacio de nombres (`targetNamespace`), porque el documento tampoco usa uno. Por eso el XML se asocia con `xsi:noNamespaceSchemaLocation` (sección 3).

### Tipos simples: qué valores se aceptan

Cada tipo simple restringe un tipo predefinido de XSD mediante *facetas*. Este schema usa `pattern`, `enumeration`, `minInclusive`, `maxInclusive`, `fractionDigits` y `minLength`.

| Tipo | Deriva de | Restricción | Se usa en | Ejemplo |
|---|---|---|---|---|
| `tipoTexto` | `xs:token` | al menos 1 carácter | nombres, estadio, ciudad, jugadores | `Estadio León` |
| `tipoTorneo` | `xs:string` | patrón `(Apertura\|Clausura) [0-9]{4}` | `liga/@torneo` | `Apertura 2026` |
| `tipoTemporada` | `xs:string` | patrón `[0-9]{4}-[0-9]{4}` | `liga/@temporada` | `2026-2027` |
| `tipoNumeroJornada` | `xs:positiveInteger` | máximo 17 | `jornada/@numero` | `10` |
| `tipoIdPartido` | `xs:string` | patrón `P[0-9]{3}` | `partido/@id` | `P001` |
| `tipoEstadoPartido` | `xs:string` | `programado`, `en_juego`, `finalizado`, `suspendido` o `pospuesto` | `partido/@estado` | `finalizado` |
| `xs:date` *(predefinido)* | — | fecha existente en formato AAAA-MM-DD (admite zona horaria opcional) | `fecha` | `2026-09-27` |
| `tipoHora` | `xs:token` | patrón `([01][0-9]\|2[0-3]):[0-5][0-9]` | `hora` | `19:00` |
| `tipoZonaHoraria` | `xs:string` | patrón `UTC[+\-](0[0-9]\|1[0-4]):(00\|30\|45)` | `hora/@zona` | `UTC-07:00` |
| `tipoAbreviatura` | `xs:string` | patrón `[A-Z]{2,4}` | `@abreviatura` | `UNAM` |
| `tipoPorcentaje` | `xs:decimal` | de 0 a 100, con un decimal como máximo | `posesion` | `54.3` |
| `xs:nonNegativeInteger` *(predefinido)* | — | entero mayor o igual que 0 | tiros, faltas, goles y demás conteos | `15` |
| `tipoMinuto` | `xs:positiveInteger` | máximo 90 | `@minuto` | `76` |
| `tipoAgregado` | `xs:positiveInteger` | máximo 30 | `@agregado` | `9` |
| `tipoColorTarjeta` | `xs:string` | `amarilla` o `roja` | `tarjeta/@color` | `amarilla` |
| `tipoLado` | `xs:string` | `local` o `visitante` | `gol/@equipo` | `local` |
| `tipoOrigenGol` | `xs:string` | `jugada`, `penal` o `autogol` | `gol/@tipo` | `penal` |

Por qué esos límites:

- **La hora no usa `xs:time`**, que exige segundos (`19:00:00`). El patrón exige el formato `HH:MM` de 24 horas y rechaza valores como `24:00` o `7:00 pm`.
- **17 jornadas de hasta 9 partidos.** En el Apertura 2026 juegan [18 equipos, todos contra todos a una vuelta, en 17 jornadas](https://www.tvazteca.com/azteca7/chismes-deportivos/como-funciona-la-liga-mx-y-quienes-juegan/). Con 18 equipos, una jornada tiene a lo más 9 partidos.
- **El minuto llega a 90.** La fase regular no tiene tiempos extra, y lo que pasa en la reposición se escribe con `agregado`. Un gol capturado como `minuto="95"` se rechaza; lo correcto es `minuto="90" agregado="5"`. Si el formato se ampliara a la liguilla, cuya final puede ir a tiempos extra, el máximo subiría a 120.
- **El tope de `agregado` es holgado a propósito.** En la práctica, ninguna mitad llega a 30 minutos de reposición. El límite sirve para detectar errores de captura, como escribir el minuto total en `agregado`.
- **`tipoTexto` deriva de `xs:token`**, que quita los espacios del inicio y del final antes de validar. Por eso ni `<nombre></nombre>` ni `<nombre>   </nombre>` son válidos.

### Tipos complejos y cardinalidad

| Elemento | Tipo | Contenido (en orden) | Veces que aparece |
|---|---|---|---|
| `liga` | `tipoLiga` | `jornada` | raíz |
| `jornada` | `tipoJornada` | `partido` | 1 a 17 |
| `partido` | `tipoPartido` | `fecha`, `hora`, `estadio`, `equipoLocal`, `equipoVisitante`, `marcador` | 1 a 9 por jornada |
| `hora` | `tipoHoraConZona` | texto (`tipoHora`) y atributo `zona` | 1 |
| `estadio` | `tipoEstadio` | texto y atributo `ciudad` | 1 |
| `equipoLocal`, `equipoVisitante` | `tipoEquipo` | `nombre`, `estadisticas` | 1 cada uno |
| `estadisticas` | `tipoEstadisticas` | `posesion`, `tiros`, `tirosAPuerta`, `faltas`, `tirosDeEsquina`, `fuerasDeLugar` (opcional), `atajadas` (opcional), `tarjetas` | 0 o 1 |
| `tarjetas` | `tipoTarjetas` | `tarjeta` | 1 |
| `tarjeta` | `tipoTarjeta` | nombre del jugador y atributos `color`, `minuto`, `agregado` | 0 o más |
| `marcador` | `tipoMarcador` | `golesLocal`, `golesVisitante`, `gol` | 0 o 1 |
| `gol` | `tipoGol` | nombre del anotador y atributos `equipo`, `minuto`, `agregado`, `tipo` | 0 o más |

- Todos los modelos de contenido son `xs:sequence`: el orden importa, igual que con las comas del DTD.
- `minOccurs` y `maxOccurs` valen 1 por omisión y solo se escriben cuando cambian. En el DTD, un límite como «de 1 a 9» solo se lograba anidando grupos opcionales, `(partido, (partido, (partido, …)?)?)`; con `maxOccurs="9"` y `maxOccurs="17"` el schema lo expresa directamente.
- El marcador es opcional porque un partido programado todavía no lo tiene.
- Los elementos que tienen texto y además atributos (`hora`, `estadio`, `tarjeta` y `gol`) se declaran con `xs:simpleContent` y `xs:extension`: a un tipo simple se le agregan atributos.

### Atributos

| Elemento | Atributo | Tipo | Uso |
|---|---|---|---|
| `liga` | `nombre`, `torneo`, `temporada` | `tipoTexto`, `tipoTorneo`, `tipoTemporada` | obligatorio |
| `jornada` | `numero` | `tipoNumeroJornada` | obligatorio |
| `partido` | `id` | `tipoIdPartido` | obligatorio y único (ver `xs:key`) |
| `partido` | `estado` | `tipoEstadoPartido` | obligatorio |
| `hora` | `zona` | `tipoZonaHoraria` | opcional; por omisión `UTC-06:00` |
| `estadio` | `ciudad` | `tipoTexto` | obligatorio |
| `equipoLocal`, `equipoVisitante` | `abreviatura` | `tipoAbreviatura` | obligatorio |
| `tarjeta` | `color` | `tipoColorTarjeta` | obligatorio |
| `tarjeta`, `gol` | `minuto` | `tipoMinuto` | obligatorio |
| `tarjeta`, `gol` | `agregado` | `tipoAgregado` | opcional; solo en tiempo de reposición |
| `gol` | `equipo` | `tipoLado` | obligatorio |
| `gol` | `tipo` | `tipoOrigenGol` | opcional; por omisión `jugada` |

Equivalencias con el DTD: `use="required"` corresponde a `#REQUIRED`; un atributo sin `use` ni `default`, a `#IMPLIED`; y `default="…"`, a un valor por omisión. Casi todos los goles son de jugada, así que `tipo` solo se escribe en penales y autogoles.

### Restricciones de identidad

| Restricción | Se declara en | Selector → campo | Qué garantiza |
|---|---|---|---|
| `xs:key idPartido` | `liga` | `jornada/partido` → `@id` | cada partido tiene un `id` distinto en todo el documento |
| `xs:unique numeroJornada` | `liga` | `jornada` → `@numero` | no hay dos jornadas con el mismo número |
| `xs:unique equipoPorJornada` | `jornada` | `partido/equipoLocal \| partido/equipoVisitante` → `@abreviatura` | un equipo juega una sola vez por jornada, y nunca contra sí mismo |

El alcance de la restricción depende del elemento donde se declara. `equipoPorJornada` se declara en `jornada`, así que Pumas puede aparecer en la jornada 10 y en la 11, pero no dos veces en la misma.

En el DTD, la unicidad del identificador venía del tipo `ID`, que vale en todo el documento y solo admite nombres XML. `xs:key` y `xs:unique` permiten elegir el alcance y el tipo del valor, y sirven también para datos que no son identificadores, como el número de jornada o la abreviatura de un equipo. La diferencia entre las dos: `xs:key` exige que el campo exista en cada elemento seleccionado y puede ser el destino de una referencia (`xs:keyref`); `xs:unique` solo exige que los valores presentes no se repitan.

## 3. Documento: jornada del domingo 27 de septiembre de 2026

`xml/resultados.xml` contiene la jornada 10 del Apertura 2026 con los tres partidos de ese domingo:

| ID | Hora | Local | Marcador | Visitante | Estadio | Ficha en ESPN |
|---|---|---|---|---|---|---|
| P001 | 12:00 | Pumas UNAM | 2–3 | Atlético de San Luis | Estadio Olímpico Universitario, Ciudad de México | [juegoId 401876958](https://www.espn.com.mx/futbol/partido/_/juegoId/401876958/atletico-de-san-luis-pumas-unam) |
| P002 | 19:00 | León | 2–1 | FC Juárez | Estadio León, León | [juegoId 401876957](https://www.espn.com.mx/futbol/partido/_/juegoId/401876957/fc-juarez-leon) |
| P003 | 21:10 | Necaxa | 2–4 | América | Estadio Victoria, Aguascalientes | [juegoId 401876956](https://www.espn.com.mx/futbol/partido/_/juegoId/401876956/america-necaxa) |

La jornada 10 se jugó del viernes 25 al domingo 27 de septiembre y tuvo nueve partidos. El documento registra los tres del domingo, que son los que pide la tarea; los otros seis cabrían sin cambios, porque el schema admite hasta nueve partidos por jornada.

De cada partido se registran el marcador; cada gol, con su minuto, su anotador y su origen (jugada, penal o autogol); siete estadísticas por equipo, y cada tarjeta con su minuto.

### Decisiones de captura

| Dato de ESPN | Cómo se registró |
|---|---|
| **Autogol** (León–Juárez, 76'). El gol cuenta para León, pero lo anotó Francisco Nevarez, de Juárez. | `equipo="local"` indica a quién se le cuenta el gol, y `tipo="autogol"` explica por qué el anotador es del otro equipo. |
| **Gol de cabeza** (Necaxa–América, 39', Henry Martín). | Se registró como `jugada`. "De cabeza" describe la técnica, no el origen del gol; ponerlo junto a `penal` y `autogol` mezclaría dos clasificaciones, porque un autogol también puede ser de cabeza. |
| **Tiempo de reposición** (45+10, 90+9). | `minuto="45" agregado="10"` y `minuto="90" agregado="9"`. |

Ejemplo: el marcador del León–Juárez, con un autogol y dos goles en tiempo de reposición, uno de ellos de penal.

```xml
<marcador>
  <golesLocal>2</golesLocal>
  <golesVisitante>1</golesVisitante>
  <gol equipo="local" minuto="76" tipo="autogol">Francisco Nevarez</gol>
  <gol equipo="visitante" minuto="90" agregado="4" tipo="penal">Óscar Estupiñán</gol>
  <gol equipo="local" minuto="90" agregado="9">José Alvarado</gol>
</marcador>
```

### Asociación con el schema

```xml
<liga xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="../xsd/resultados.xsd"
      nombre="Liga MX" torneo="Apertura 2026" temporada="2026-2027">
```

- `xmlns:xsi` declara el espacio de nombres *XML Schema Instance*, al que pertenece el atributo `noNamespaceSchemaLocation`.
- `xsi:noNamespaceSchemaLocation` indica dónde está el schema de los elementos que no tienen espacio de nombres. Si el schema declarara un `targetNamespace`, se usaría `xsi:schemaLocation` con el par «espacio de nombres – ruta».
- La ruta es relativa **al archivo XML**: desde `xml/` se sube a `liga-mx-xml/` y se entra a `xsd/`.
- El documento ya no lleva `DOCTYPE`. La referencia al schema es una indicación para las herramientas: los editores la siguen solos, y en la línea de comandos el schema se indica explícitamente.

## 4. Validación

### Cómo validar

Desde la carpeta `liga-mx-xml/`:

**xmllint** (incluido en macOS; en Debian y Ubuntu viene en el paquete `libxml2-utils`; en Windows funciona desde WSL):

```bash
xmllint --noout --schema xsd/resultados.xsd xml/resultados.xml
```

`--schema` indica el XSD contra el que se valida, y `--noout` evita que se imprima el documento. Si el documento es válido, xmllint responde `xml/resultados.xml validates`. Si no, lista cada error con su número de línea y termina con `fails to validate`.

**VS Code** (con la extensión *XML* de Red Hat) o **IntelliJ IDEA**: siguen `xsi:noNamespaceSchemaLocation`, validan al abrir el archivo, subrayan los errores y autocompletan elementos y atributos según el schema.

**Python** con `lxml` (`pip install lxml`; en macOS y Linux el comando suele ser `python3`):

```bash
python -c "from lxml import etree; s = etree.XMLSchema(etree.parse('xsd/resultados.xsd')); s.assertValid(etree.parse('xml/resultados.xml')); print('válido')"
```

### Resultado

```text
$ xmllint --noout --schema xsd/resultados.xsd xml/resultados.xml
xml/resultados.xml validates
```

Otros dos validadores independientes también aceptan el documento: el que incluye Java (`javax.xml.validation`, basado en Apache Xerces) y la biblioteca `xmlschema` de Python. Además, se contrastaron todos los valores con la ficha de ESPN de cada partido (marcador, goles detallados por equipo, las siete estadísticas numéricas y las tarjetas) y se revisaron aparte las reglas entre valores que el schema no puede expresar (sección 6).

### Pruebas con datos inválidos

`xml/resultados-invalido.xml` contiene diez pruebas y cada una tiene **un solo error**, así que una sola ejecución muestra los diez diagnósticos. La última columna indica si el DTD del ejercicio anterior habría detectado el mismo error.

| # | Prueba | Regla que se viola | Mensaje de xmllint (resumido) | ¿Lo detectaba el DTD? |
|---|---|---|---|---|
| 1 | Falta el visitante | secuencia de `tipoPartido` | `Element 'marcador': This element is not expected. Expected is ( equipoVisitante )` | Sí |
| 2 | `<tiros>siete</tiros>` | `xs:nonNegativeInteger` | `'siete' is not a valid value of the atomic type 'xs:nonNegativeInteger'` | No |
| 3 | `<posesion>533</posesion>` (53.3 sin el punto) | `maxInclusive` 100 | `The value '533' is greater than the maximum value allowed ('100')` | No |
| 4 | Pumas juega dos veces en la jornada 10 | `xs:unique equipoPorJornada` | `Duplicate key-sequence ['UNAM'] in unique identity-constraint 'equipoPorJornada'` | No |
| 5 | `<fecha>04/10/2026</fecha>` | `xs:date` | `'04/10/2026' is not a valid value of the atomic type 'xs:date'` | No |
| 6 | `<hora>7:00 pm</hora>` | patrón de `tipoHora` | `The value '7:00 pm' is not accepted by the pattern …` | No |
| 7 | Gol con `minuto="95"` | `maxInclusive` 90 | `The value '95' is greater than the maximum value allowed ('90')` | No |
| 8 | `<nombre></nombre>` | `minLength` 1 | `The value has a length of '0'; this underruns the allowed minimum length of '1'` | No |
| 9 | Segundo partido con `id="P101"` | `xs:key idPartido` | `Duplicate key-sequence ['P101'] in key identity-constraint 'idPartido'` | Sí (tipo `ID`) |
| 10 | Dos jornadas con `numero="10"` | `xs:unique numeroJornada` | `Duplicate key-sequence ['10'] in unique identity-constraint 'numeroJornada'` | No |

Salida completa:

```text
$ xmllint --noout --schema xsd/resultados.xsd xml/resultados-invalido.xml
xml/resultados-invalido.xml:23: element marcador: Schemas validity error : Element 'marcador': This element is not expected. Expected is ( equipoVisitante ).
xml/resultados-invalido.xml:38: element tiros: Schemas validity error : Element 'tiros': 'siete' is not a valid value of the atomic type 'xs:nonNegativeInteger'.
xml/resultados-invalido.xml:62: element posesion: Schemas validity error : Element 'posesion': [facet 'maxInclusive'] The value '533' is greater than the maximum value allowed ('100').
xml/resultados-invalido.xml:87: element equipoVisitante: Schemas validity error : Element 'equipoVisitante': Duplicate key-sequence ['UNAM'] in unique identity-constraint 'equipoPorJornada'.
xml/resultados-invalido.xml:97: element fecha: Schemas validity error : Element 'fecha': '04/10/2026' is not a valid value of the atomic type 'xs:date'.
xml/resultados-invalido.xml:111: element hora: Schemas validity error : Element 'hora': [facet 'pattern'] The value '7:00 pm' is not accepted by the pattern '([01][0-9]|2[0-3]):[0-5][0-9]'.
xml/resultados-invalido.xml:136: element gol: Schemas validity error : Element 'gol', attribute 'minuto': [facet 'maxInclusive'] The value '95' is greater than the maximum value allowed ('90').
xml/resultados-invalido.xml:149: element nombre: Schemas validity error : Element 'nombre': [facet 'minLength'] The value has a length of '0'; this underruns the allowed minimum length of '1'.
xml/resultados-invalido.xml:154: element partido: Schemas validity error : Element 'partido': Duplicate key-sequence ['P101'] in key identity-constraint 'idPartido'.
xml/resultados-invalido.xml:169: element jornada: Schemas validity error : Element 'jornada': Duplicate key-sequence ['10'] in unique identity-constraint 'numeroJornada'.
xml/resultados-invalido.xml fails to validate
```

Cada error aparece en la línea de su prueba, y xmllint termina con código de salida 3 (error de validación). lxml, `xmlschema` y el validador de Java detectan los mismos diez errores.

También se probó lo contrario: los valores en el límite sí son válidos. Entre ellos están la posesión 0 y 100, el minuto 1 y el minuto 90 con 30 de agregado, una jornada con 9 partidos, 17 jornadas y un mismo equipo en dos jornadas distintas.

> **Bien formado no es lo mismo que válido.** `resultados-invalido.xml` respeta la sintaxis de XML (`xmllint --noout xml/resultados-invalido.xml` no reporta nada); lo que viola son las reglas del schema, y eso solo lo revisa un validador.

## 5. DTD frente a XSD

El ejercicio anterior validaba el mismo documento con un DTD. Estas son las diferencias:

| Regla | DTD | XSD |
|---|---|---|
| Orden y presencia de los elementos | Sí | Sí |
| Límites de repetición (1 a 9 partidos) | Solo anidando grupos opcionales (poco práctico) | Sí: `minOccurs` y `maxOccurs` |
| Tipos de datos (entero, decimal, fecha) | No: todo es texto | Sí |
| Rangos (posesión de 0 a 100, minuto de 1 a 90) | No | Sí |
| Formatos (hora `HH:MM`, id `P###`) | No | Sí: `pattern` |
| Texto obligatorio no vacío | No | Sí: `minLength` |
| Unicidad | Solo `ID`, en todo el documento | `xs:key` y `xs:unique`, con el alcance que se elija |
| Reutilizar definiciones | Repetir o usar entidades | Tipos con nombre y grupos de atributos |
| Sintaxis | Propia | XML: se lee y se procesa con las mismas herramientas |
| Relaciones entre valores (goles = número de `gol`) | No | No en XSD 1.0; sí en XSD 1.1 |

## 6. Límites del XSD 1.0

El schema valida cada valor por separado. Entre valores distintos, XSD 1.0 solo puede exigir que no se repitan (`xs:key`, `xs:unique`) o que uno haga referencia a otro (`xs:keyref`); no puede expresar sumas, comparaciones ni condiciones. Por eso estas reglas quedan fuera:

- `golesLocal` y `golesVisitante` coinciden con el número de `gol` de cada lado, cuando los goles están detallados.
- Las dos posesiones de un partido suman 100.
- `tirosAPuerta` no supera a `tiros`.
- Un partido `finalizado` tiene `marcador`, y uno `programado` no.
- `agregado` solo acompaña a los minutos 45 y 90.
- Los años de `temporada` son consecutivos y coinciden con el año del `torneo`.

XSD 1.1 agrega aserciones (`xs:assert`): condiciones escritas en XPath que el validador evalúa. Por ejemplo, en `tipoMarcador`:

```xml
<xs:assert test="not(gol) or (golesLocal = count(gol[@equipo = 'local'])
                              and golesVisitante = count(gol[@equipo = 'visitante']))"/>
```

Con esa línea, la biblioteca `xmlschema`, que implementa XSD 1.1, rechaza un marcador 3–3 que solo detalla dos goles locales. No se incluyó en `resultados.xsd` porque xmllint, lxml, el validador de Java y los editores más comunes solo implementan XSD 1.0, y con ella el schema ni siquiera compilaría. Por eso estas seis reglas se revisaron por separado, al contrastar el documento con ESPN.
