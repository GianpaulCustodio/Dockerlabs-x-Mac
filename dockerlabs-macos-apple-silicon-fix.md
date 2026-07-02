# Solucionar DockerLabs (`auto_deploy.sh`) en macOS Apple Silicon (M1/M2/M3/M4)

Guía para arreglar los dos problemas más comunes al correr el script `auto_deploy.sh` de [DockerLabs](https://dockerlabs.es/) en un Mac con chip Apple Silicon:

1. Los colores ANSI no se ven y en su lugar aparece texto literal como `\e[0m`, `\e[1;93m`, etc.
2. La máquina se despliega correctamente y te da una IP (ej. `172.17.0.2`), pero el `ping` a esa IP nunca responde.

Ambos problemas son de macOS, no del script en sí — en Linux `auto_deploy.sh` funciona sin tocar nada.

---

## Requisitos previos

- [Homebrew](https://brew.sh) instalado.
- [Docker Desktop para Mac](https://www.docker.com/products/docker-desktop) instalado y **abierto** (la ballena en la barra de menú debe estar quieta, sin animación de arranque).

Si no tienes Docker Desktop:

```bash
brew install --cask docker
open -a Docker
```

Espera a que termine de iniciar la VM interna antes de continuar.

---

## Problema 1: se imprime `\e[0m` en vez del color

### Causa

macOS trae por defecto **bash 3.2** (Apple lo congeló en esa versión desde 2007 por el cambio de licencia a GPLv3 y no lo ha actualizado desde entonces). El builtin `echo -e` de bash 3.2 **no reconoce `\e` como código de escape ANSI**, solo entiende la notación octal `\033`. Como muchos scripts de DockerLabs están escritos/probados en Linux con bash 4/5 (donde `\e` sí funciona), al correrlos en macOS el `\e[1;93m` se imprime como texto plano en vez de convertirse en color.

Puedes confirmar tu versión de bash con:

```bash
bash --version
# GNU bash, version 3.2.57(1)-release  <-- la que trae macOS por defecto
```

### Solución

Reemplaza todas las apariciones de `\e[` por `\033[` en el script. Es un cambio de una sola línea con `sed` (en macOS `sed -i` necesita el argumento vacío `''` después de `-i`):

```bash
sed -i '' 's/\\e\[/\\033[/g' auto_deploy.sh
```

Verifica que no quede ningún `\e[` suelto y que la sintaxis del script siga siendo válida:

```bash
grep -n '\\e\[' auto_deploy.sh   # no debería devolver nada
bash -n auto_deploy.sh           # "sin salida" = sintaxis OK
```

Con esto, todos los `echo -e "\033[...]"` funcionan igual en bash 3.2, 4.x y 5.x, así que el script queda portable entre Linux y macOS.

> Nota: si al abrir el script ves texto raro tipo `mÃ¡quina` en vez de `máquina`, es un problema de doble codificación UTF-8 (mojibake), no relacionado con lo anterior. Se corrige reabriendo/guardando el archivo forzando UTF-8, o con un editor que detecte la codificación real.

---

## Problema 2: la máquina se despliega pero el `ping` a su IP no responde

### Causa

Docker Desktop en macOS **no corre nativamente**: levanta una VM Linux ligera por debajo (LinuxKit) y ahí es donde realmente vive el daemon de Docker y la red bridge de los contenedores (`172.17.0.0/16` por defecto). En Linux, esa red bridge (`docker0`) es visible directamente desde el host porque el host *es* el mismo kernel Linux. En macOS, esa red vive **dentro de la VM**, y el host (tu Mac) no tiene ninguna ruta hacia ella — por eso el `ping` nunca contesta, aunque el contenedor esté sano y corriendo.

Esto **no es un bug del script**: es una limitación estructural de Docker Desktop en macOS.

### Solución: `docker-mac-net-connect`

[`docker-mac-net-connect`](https://github.com/chipmk/docker-mac-net-connect) es una herramienta de código abierto que crea un túnel WireGuard entre tu Mac y la VM de Docker, e inyecta automáticamente las rutas necesarias para que puedas hacer `ping`/conectarte directo a las IPs de los contenedores, igual que en Linux.

#### Instalación

```bash
brew install chipmk/tap/docker-mac-net-connect
sudo brew services start chipmk/tap/docker-mac-net-connect
```

El servicio debe correr como **root** (`sudo`) porque necesita permisos para crear la interfaz de red virtual (`utun`) y modificar la tabla de rutas del sistema.

#### Aprobar la extensión de red (primera vez)

La primera vez que arranca, macOS puede pedir aprobar una **Network Extension**. Si no aparece el aviso o lo cerraste sin querer, revísalo manualmente en:

**Ajustes del Sistema → General → Login Items y Extensiones → Extensiones de Red**

Busca `docker-mac-net-connect` y confirma que el interruptor esté activado. Sin esta aprobación, el servicio se queda "started" pero nunca establece el túnel de verdad.

#### Orden de arranque importante

`docker-mac-net-connect` necesita que **Docker Desktop ya esté completamente arrancado** antes de conectarse, y además escanea las redes de Docker existentes al iniciar — si un contenedor o su red se crea *después*, puede que no la detecte hasta reiniciar el servicio. El orden correcto es:

1. Abrir Docker Desktop y esperar a que la VM esté lista.
2. Desplegar el contenedor (`sudo bash auto_deploy.sh nombre.tar`).
3. Iniciar o reiniciar `docker-mac-net-connect`.

```bash
sudo brew services restart chipmk/tap/docker-mac-net-connect
```

#### Verificación

```bash
# 1. Confirmar que el binario está instalado
which docker-mac-net-connect

# 2. Confirmar que el servicio está corriendo
sudo brew services list | grep docker-mac-net-connect

# 3. Buscar la interfaz WireGuard que crea la herramienta
#    (aparece como utun con una IP tipo 10.x.x.1 --> 10.x.x.2 y netmask /32)
ifconfig | grep -A2 utun

# 4. Confirmar que existe una ruta hacia la red de Docker
netstat -rn -f inet | grep utun
```

Si el paso 4 no muestra ninguna ruta hacia `172.17.x.x` (o el rango que use tu red de Docker), reinicia el servicio con Docker y el contenedor ya arriba (ver "Orden de arranque importante").

> macOS puede acumular interfaces `utun` viejas de otras VPNs (Tailscale, WireGuard, VPNs corporativas, etc.) sin IPv4 asignada, solo con una dirección `fe80::...` (IPv6 link-local). Esas se ignoran — la que te interesa es la que tiene una IP `10.x.x.x` punto a punto.

#### Prueba final

```bash
ping 172.17.0.2   # (usa la IP que te dio auto_deploy.sh)
```

---

## Checklist rápido

- [ ] Docker Desktop instalado y abierto antes de correr `auto_deploy.sh`
- [ ] `sed -i '' 's/\\e\[/\\033[/g' auto_deploy.sh` aplicado (colores correctos)
- [ ] `docker-mac-net-connect` instalado (`brew install chipmk/tap/docker-mac-net-connect`)
- [ ] Servicio iniciado con `sudo` y con Docker Desktop ya arriba
- [ ] Extensión de red aprobada en Ajustes del Sistema
- [ ] `ifconfig | grep -A2 utun` muestra una interfaz con IP `10.x.x.x`
- [ ] `netstat -rn -f inet | grep utun` muestra ruta hacia la red del contenedor
- [ ] `ping <IP del contenedor>` responde

---

## Alternativa sin `docker-mac-net-connect`

Si no quieres instalar nada adicional (o la extensión de red te da problemas), puedes publicar los puertos del contenedor en vez de acceder por IP directa, aunque esto cambia la forma de practicar el escaneo de puertos típico de estos labs:

```bash
docker run -P -d --name mi_contenedor mi_imagen
docker port mi_contenedor
```

Luego te conectas por `127.0.0.1:<puerto publicado>` en vez de la IP interna del contenedor.

---

## Resumen técnico

| Problema | Causa raíz | Solución |
|---|---|---|
| `\e[0m` se imprime literal | bash 3.2 de macOS no soporta `\e` en `echo -e`, solo `\033` | Reemplazar `\e[` por `\033[` en el script |
| `ping` a la IP del contenedor no responde | Docker Desktop en macOS corre en una VM aislada; el host no tiene ruta a la red bridge interna | Instalar y correr `docker-mac-net-connect`, con Docker y el contenedor ya arriba antes de iniciar el servicio |

Probado en macOS con chip Apple Silicon (M3) y Docker Desktop para Mac.
