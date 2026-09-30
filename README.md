# robot-ecosystem

Repo **paraguas**: agrupa los proyectos que hacen que los robots Unitree sean operables y
observables desde fuera de su propia subred. No contiene código de los proyectos — cada uno
es un repo independiente (su propio `.git`) y está gitignoreado acá, igual que en
`AI-VL-ecosystem`.

## Los cuatro repos

```
                        ┌─────────────────────────── EN EL ROBOT (Jetson) ──┐
  rt/lf/lowstate ──DDS──▶ robot-telemetry-agent ──HTTPS──▶ Splunk HEC       │
                        │   solo lectura, no puede mover el robot           │
                        │                                                   │
  AI-VL executor ─HTTP──▶ robot-command-relay ──DDS──▶ 🤖                   │
                        │   token, allowlist, clamp, hombre muerto          │
                        │                                                   │
  cámara ─────────DDS───▶ robot-video-pipeline ─RTMP─┐                      │
                        └───────────────────────────┼───────────────────────┘
                                                    ▼
                                    mediamtx ──▶ Frigate (grabación)
                                             ├──▶ navegador (HLS/WebRTC)
                                             └──▶ AI-VL y Splunk (MJPEG, baja latencia)
```

| Repo | Qué hace | Dónde corre |
|---|---|---|
| **`robot-video-pipeline`** | Saca la cámara del DDS y la convierte en video estándar: vista en vivo de baja latencia, streaming y grabación en NVR | Mitad en el robot (encode por hardware), mitad en el servidor (mediamtx + Frigate) |
| **`robot-telemetry-agent`** | Lee el estado por DDS, extrae campos y los manda a Splunk por HTTPS. Solo lectura | En el robot |
| **`robot-command-relay`** | El único camino remoto que puede mover al robot. Token, rate limit, allowlist, clamp de velocidad, hombre muerto | En el robot |
| **`robot-splunk-docs`** | El registro de ingeniería: mediciones, decisiones y planes que citan los otros repos | No corre |

## Go2 y G1: qué va en cada robot

Los tres repos del robot sirven a los dos robots. Qué archivo corre en cuál, y la convención
de nombres que lo hace visible (`go2_*`, `g1_*` / `host/g1/`, sin prefijo = compartido), están
en **`robot-splunk-docs/QUE-CORRE-EN-CADA-ROBOT.md`**.

## La restricción que explica el diseño

El DDS de estos robots no cruza una frontera de subred. Medido en un Go2: **122** tópicos
visibles desde su propia subred, **2** desde otra con ruteo funcionando (ping de 1,3 ms), y
**3** incluso con peers unicast explícitos.

De ahí sale la forma de todo lo de acá: **DDS corto y local, HTTPS largo y ruteado.** Lo que
tiene que estar pegado al DDS corre en el robot; lo que cruza la red es HTTP. El razonamiento
completo, con las mediciones, está en `robot-splunk-docs/RED-Y-DDS.md`.

## Relación con AI-VL

AI-VL es el consumidor, no parte de esto. Le habla a este ecosistema **solo por HTTP**:

- su ejecutor (`unitree_ros2/robot_executor/`) le pega a `robot-command-relay` cuando el
  transporte de un robot está en modo `relay`;
- su puente de cámara (`unitree_ros2/robot_camera_bridge/`) puede leer el MJPEG que sirve
  `robot-video-pipeline` en lugar de leer DDS, que es lo que funciona cuando el robot ya no
  comparte subred.

## Clonar

Los cuatro van **adentro de esta carpeta**, manteniendo los nombres:

```bash
git clone https://github.com/Maxi-Andres/robot-video-pipeline.git   robot-video-pipeline
git clone https://github.com/Maxi-Andres/robot-telemetry-agent.git  robot-telemetry-agent
git clone https://github.com/Maxi-Andres/robot-command-relay.git    robot-command-relay
git clone https://github.com/Maxi-Andres/robot-splunk-docs.git      robot-splunk-docs
```

## Dependencia común

Los tres repos con código C++ compilan contra el SDK oficial, que vive **afuera** de este
paraguas y sin parchar:

```bash
git clone https://github.com/unitreerobotics/unitree_sdk2.git ~/Desktop/unitree_sdk2
```

Los `build.sh` lo buscan en `$UNITREE_SDK2_DIR`, `~/unitree_sdk2` y `~/Desktop/unitree_sdk2`,
en ese orden.

## Convenciones

- **Código y comentarios en inglés**, sin excepción. Los documentos de planificación de
  `robot-splunk-docs` van en castellano a propósito.
- **Nunca `git commit` ni `git push`**: el usuario revisa y commitea a mano.
