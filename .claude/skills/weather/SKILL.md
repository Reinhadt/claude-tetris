---
name: weather
description: Obtiene el clima local actual (o el de una ciudad indicada) usando wttr.in por linea de comandos, sin necesidad de API key. Usar cuando el usuario pregunte por el clima, temperatura, pronostico o condiciones meteorologicas de su ubicacion actual o de una ciudad especifica.
---

# Clima local

Obtiene datos del clima usando el servicio publico [wttr.in](https://wttr.in), que no requiere API key. Si no se especifica ciudad, wttr.in detecta la ubicacion automaticamente a partir de la IP de salida del entorno.

## Uso rapido

Resumen en una linea (ideal para respuestas cortas):

```bash
curl -s --max-time 5 "wttr.in/?format=3"
```

Salida de ejemplo: `León, Guanajuato, MX: ☀️  +23°C`

Para una ciudad especifica, reemplaza el espacio en blanco por `+`:

```bash
curl -s --max-time 5 "wttr.in/Ciudad+de+Mexico?format=3"
```

## Reporte detallado (ASCII, 3 dias de pronostico)

```bash
curl -s --max-time 5 "wttr.in"
```

Para forzar unidades metricas y evitar secuencias de color ANSI (mas facil de leer en logs):

```bash
curl -s --max-time 5 "wttr.in?m&T"
```

## Datos estructurados (JSON)

Cuando se necesiten campos especificos (temperatura, sensacion termica, humedad, viento, probabilidad de lluvia, etc.) para procesarlos o presentarlos de forma personalizada:

```bash
curl -s --max-time 5 "wttr.in/?format=j1"
```

Campos utiles dentro del JSON:
- `current_condition[0].temp_C` / `temp_F` — temperatura actual
- `current_condition[0].FeelsLikeC` — sensacion termica
- `current_condition[0].humidity` — humedad (%)
- `current_condition[0].weatherDesc[0].value` — descripcion textual
- `current_condition[0].windspeedKmph` — viento
- `weather[0].hourly[].chanceofrain` — probabilidad de lluvia por hora (dia actual)

## Notas

- Siempre usa `--max-time 5` para no bloquear si el servicio no responde.
- Si el comando falla o no hay conexion a internet, informa al usuario en vez de inventar datos del clima.
- Si el usuario pide el clima de una ciudad con espacios o caracteres especiales, reemplaza los espacios por `+` y evita caracteres que rompan la URL (usa `curl --get --data-urlencode` si el nombre tiene acentos o simbolos raros).
- Presenta la respuesta al usuario en lenguaje natural (no pegues el JSON crudo) salvo que pida los datos en bruto.
