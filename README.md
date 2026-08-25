# Agente HERMES — Chat de Preguntas y Respuestas con Kimi AI

Agente de IA que corre en una Raspberry Pi dentro de una unidad HERMES.
Funciona como un **chat de preguntas y respuestas**: el usuario escribe un
mensaje, el agente lo envía al LLM de **Kimi AI (Moonshot AI)** y devuelve
la respuesta. Además puede consultar el estado del propio equipo (CPU,
temperatura, RAM) si el LLM lo necesita para responder.

## Qué necesitas antes de instalar

1. Una Raspberry Pi con Raspberry Pi OS (o cualquier Linux con Python 3.11+).
2. Una **API key de Kimi AI / Moonshot AI**. Se obtiene en la plataforma de
   Moonshot AI (https://platform.moonshot.cn) creando una cuenta y generando
   una clave en la sección de API Keys.
3. Conexión a internet en la Raspberry Pi (el agente habla con la API de
   Kimi por HTTPS).

## Instalación paso a paso (Raspberry Pi)

```bash
# 1. Descarga el proyecto en la Raspberry Pi
git clone <URL-de-este-repo> hermes-agent
cd hermes-agent

# 2. Ejecuta el instalador (crea usuario de sistema, entorno virtual y
#    el servicio de systemd para que arranque solo con la Pi)
sudo bash scripts/install.sh

# 3. Configura tu API key de Kimi
sudo nano /etc/hermes-agent/hermes-agent.env
```

Dentro de ese archivo, edita como mínimo estas dos líneas:

```
KIMI_API_KEY=sk-tu-api-key-de-moonshot-ai
HERMES_ID=hermes-unit-001
```

Guarda el archivo (Ctrl+O, Enter, Ctrl+X en `nano`) y luego arranca el
servicio:

```bash
sudo systemctl start hermes-agent
sudo systemctl status hermes-agent      # confirma que está "active (running)"
sudo journalctl -u hermes-agent -f      # ver los logs en vivo
```

Con esto el agente ya queda funcionando en segundo plano y se reinicia
solo si la Pi se reinicia o si el proceso falla.

## Cómo se usa (chat de preguntas y respuestas)

El agente lee mensajes de la entrada estándar (stdin) y responde usando
Kimi AI. Para probarlo de forma interactiva (fuera del servicio, útil para
verificar que la conexión con Kimi funciona):

```bash
cd hermes-agent
source venv/bin/activate      # si instalaste con scripts/install.sh, el venv está en /opt/hermes-agent/venv
python -m hermes_agent.main
```

Luego simplemente escribe una pregunta y presiona Enter, por ejemplo:

```
> Hola, ¿qué puedes hacer?
> ¿Cuál es la temperatura del CPU en este momento?
```

El agente responderá usando el modelo de Kimi configurado (por defecto
`moonshot-v1-128k`), y si la pregunta requiere datos del propio equipo
(temperatura, uso de CPU/RAM, etc.) los consulta automáticamente antes de
responder.

## Probar en una computadora normal (sin Raspberry Pi)

No hace falta tener la Pi físicamente para probar el chat:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
pip install -e .

cp .env.example .env
nano .env    # pon tu KIMI_API_KEY y un HERMES_ID cualquiera, ej: hermes-dev

python -m hermes_agent.main
```

Fuera de una Raspberry Pi real, el módulo de hardware se simula
automáticamente (no falla por no tener GPIO), así que el chat con Kimi
funciona igual.

## Configuración disponible

Todo se configura por variables de entorno (ver `.env.example`).
Las obligatorias son:

| Variable       | Qué es                                  |
|----------------|------------------------------------------|
| `KIMI_API_KEY` | Tu API key de Moonshot AI (Kimi)          |
| `HERMES_ID`    | Nombre/identificador de este equipo HERMES |

Las demás tienen valores por defecto razonables (modelo de Kimi, reintentos
de red, tamaño de la memoria de conversación, etc.) y no es obligatorio
tocarlas para que el chat funcione.

## Comandos útiles del servicio

```bash
sudo systemctl restart hermes-agent   # reiniciar el agente
sudo systemctl stop hermes-agent      # detenerlo
sudo journalctl -u hermes-agent -f    # ver logs en vivo
```

## Estructura del proyecto (para quien quiera modificar el código)

```
hermes_agent/
  config.py             # configuración (lee KIMI_API_KEY, HERMES_ID, etc.)
  kimi_client.py         # cliente async que habla con la API de Kimi AI
  hardware_interface.py  # lectura de CPU/RAM/temperatura (con modo simulado)
  tools.py                # funciones que Kimi puede invocar (consultar hardware)
  memory.py               # historial de la conversación (SQLite)
  agent_core.py            # el "cerebro": recibe preguntas, llama a Kimi, responde
  main.py                   # punto de entrada del programa
systemd/hermes-agent.service   # definición del servicio de arranque automático
scripts/install.sh              # instalador para Raspberry Pi
tests/                            # pruebas automatizadas
```

## Si algo falla

- **"No puedo conectar con Kimi"**: revisa que `KIMI_API_KEY` esté bien
  copiada en `/etc/hermes-agent/hermes-agent.env` y que la Pi tenga
  internet. El agente reintenta automáticamente si hay cortes de red.
- **El servicio no arranca**: revisa `sudo journalctl -u hermes-agent -e`
  para ver el error exacto.
