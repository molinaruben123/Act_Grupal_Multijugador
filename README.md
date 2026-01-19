# HexMine documentación

---

## Tabla de Contenidos

1. [Descripción General](#descripción-general)
2. [Sistema de Networking](#sistema-de-networking)
3. [Integración con Steam](#integración-con-steam)
4. [Lógica de Blueprints (Detallada)](#lógica-de-blueprints-detallada)
5. [Configuración Técnica](#configuración-técnica)
8. [Inicio Rápido](#inicio-rápido)

---

## Descripción General

HexMine es un juego de disparos en primera persona (FPS) multijugador donde dos equipos (Rojo y Azul) compiten por puntos capturando minas en el mapa. El proyecto utiliza:

- **Steam Sockets** para networking de baja latencia
- **Advanced Sessions** para gestión de sesiones Steam
- **Enhanced Input** para controles modernos
- **Lumen** para iluminación global dinámica
- **Ray Tracing** habilitado

---

## Sistema de Networking

### Arquitectura

El proyecto implementa una arquitectura **Cliente-Servidor** utilizando Steam como backend de networking:

```
┌─────────────────────────────────────────────────────────────────┐
│                         STEAM BACKEND                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Steam Relay Servers (SDR)                   │   │
│  │         NAT Traversal automático + Encriptación          │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │ Cliente  │         │ HOST     │         │ Cliente  │
   │ (Steam)  │◄───────►│ (Server) │◄───────►│ (Steam)  │
   │          │         │          │         │          │
   │ BP_Fps   │         │ BP_Fps   │         │ BP_Fps   │
   │ IsHost=F │         │ IsHost=T │         │ IsHost=F │
   └──────────┘         └──────────┘         └──────────┘
```

### Configuración de NetDrivers

**Archivo:** `Config/DefaultEngine.ini`

```ini
[/Script/Engine.GameEngine]
!NetDriverDefinitions=ClearArray
+NetDriverDefinitions=(DefName="GameNetDriver",
    DriverClassName="/Script/SteamSockets.SteamSocketsNetDriver",
    DriverClassNameFallback="/Script/SteamSockets.SteamNetSocketsNetDriver")
+NetDriverDefinitions=(DefName="GameNetDriver",
    DriverClassName="OnlineSubsystemSteam.SteamNetDriver",
    DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")
```

---

## Integración con Steam

### Plugins Habilitados

| Plugin | Estado | Propósito |
|--------|--------|-----------|
| `SteamSockets` | **Habilitado** | Networking de baja latencia via Steam Datagram Relay |
| `OnlineSubsystemSteam` | Implícito | Subsistema online de Steam |
| `AdvancedSessions` | Implícito | Gestión avanzada de sesiones (via GameInstance) |

### Configuración OnlineSubsystem

**Archivo:** `Config/DefaultEngine.ini`

```ini
[OnlineSubsystem]
DefaultPlatformService=Steam

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480              ; App ID de desarrollo (Spacewar)
bInitServerOnClient=true       ; Inicializar servidor en cliente (para sesiones)

[/Script/OnlineSubsystemSteam.SteamNetDriver]
NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
```

### App ID de Steam

El proyecto usa `480` (Spacewar) para desarrollo. Para producción:

1. Crear un App ID en [Steamworks](https://partner.steamgames.com/)
2. Crear archivo `steam_appid.txt` en la raíz del build con tu App ID
3. Actualizar `SteamDevAppId` en DefaultEngine.ini

### Flujo de Conexión Multijugador

```
┌─────────────────┐
│  Inicio Juego   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────────────────────────┐
│ Steam Running?  │────►│ BP_SteamSessionInstance Inicializa  │
└────────┬────────┘     │ - OnlineSubsystemSteam activo       │
         │              │ - SteamSocketsNetDriver listo       │
         ▼              └─────────────────────────────────────┘
┌─────────────────┐
│  HOST o CLIENT  │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌────────┐
│ HOST  │ │ CLIENT │
└───┬───┘ └───┬────┘
    │         │
    ▼         ▼
┌─────────┐ ┌────────────────┐
│ Create  │ │ Find/Join      │
│ Session │ │ Session        │
└────┬────┘ └───────┬────────┘
     │              │
     ▼              ▼
┌─────────────────────────────┐
│  Conexión via Steam Sockets │
│  - NAT Traversal automático │
│  - Relay servers si falla   │
│  - Encriptación integrada   │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│   GM_Fps spawns BP_Fps      │
│   para cada jugador         │
└─────────────────────────────┘
```

---

## Lógica-de-blueprints-detallada

### BP_SteamSessionInstance (Game Instance) | **Clase Padre** | `AdvancedFriendsGameInstance` |

Este Blueprint hereda de `AdvancedFriendsGameInstance` del plugin **Advanced Sessions**, proporcionando:

- Integración automática con la lista de amigos de Steam
- Gestión de invitaciones de juego
- Manejo de sesiones online
- Callbacks para eventos de Steam (invites, join requests)
- Persistencia entre cambios de nivel

---

### GM_Fps (Game Mode) | **Clase Padre** | `GameModeBase` |

**Variables:**
| Variable | Tipo | Editable | Propósito |
|----------|------|----------|-----------|
| `RedCount` | `int32` | Sí | Contador de puntos equipo rojo |
| `BlueCount` | `int32` | Sí | Contador de puntos equipo azul |
| `RedFps` | `BP_Fps_C*` | Sí | Referencia al jugador rojo |
| `BlueFps` | `BP_Fps_C*` | Sí | Referencia al jugador azul |

**Flujo de Spawn:**
```
Jugador Conecta
       │
       ▼
GetDefaultPawnClassForController(InController)
       │
       ▼
Retorna BP_Fps (o variante según lógica)
       │
       ▼
SpawnDefaultPawnFor ejecutado
       │
       ▼
BP_Fps spawneado en PlayerStart
       │
       ▼
Asignar a RedFps o BlueFps
```

---

### BP_Fps (Character Principal) | **Clase Padre** | `Character` |

**Variables:**
| Variable | Tipo | Editable | Propósito |
|----------|------|----------|-----------|
| `IsHost` | `bool` | Sí | **REPLICADA** - Identifica si es el host |
| `RedCount` | `int32` | Sí | Puntos del equipo rojo (copia local) |
| `BlueCount` | `int32` | Sí | Puntos del equipo azul (copia local) |
| `GameMode` | `GM_Fps_C*` | Sí | Referencia al Game Mode |

**Replicación de Red - Variable IsHost:**
```
┌─────────────────────────────────────────────────────────────┐
│                     SERVIDOR (HOST)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  BP_Fps (Servidor)                                   │   │
│  │  IsHost = true                                       │   │
│  │  [Autoridad sobre esta variable]                     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                    Replicación automática
                              │
         ┌────────────────────┴────────────────────┐
         ▼                                         ▼
┌─────────────────┐                       ┌─────────────────┐
│    CLIENTE 1    │                       │    CLIENTE 2    │
│ ┌─────────────┐ │                       │ ┌─────────────┐ │
│ │ BP_Fps      │ │                       │ │ BP_Fps      │ │
│ │ IsHost = F  │ │                       │ │ IsHost = F  │ │
│ │             │ │                       │ │             │ │
│ │ OnRep_IsHost│ │                       │ │ OnRep_IsHost│ │
│ │ se ejecuta  │ │                       │ │ se ejecuta  │ │
│ └─────────────┘ │                       │ └─────────────┘ │
└─────────────────┘                       └─────────────────┘
```

La función `OnRep_IsHost` se ejecuta automáticamente en clientes cuando el valor de `IsHost` cambia, permitiendo:
- Activar/desactivar lógica específica del host
- Actualizar UI basada en el rol del jugador
- Sincronizar estado visual entre servidor y clientes

---

### BP_Mine (Actor de Mina) | **Clase Padre** | `Actor` |

**Ciclo de Vida (6 eventos):**
```
┌──────────────┐
│  BeginPlay   │ ──► Obtener referencia a GM_Fps
└──────┬───────┘     (Get Game Mode → Cast to GM_Fps)
       │
       ▼
┌──────────────┐
│  En Escena   │ ◄─┐ Loop de gameplay
└──────┬───────┘   │
       │           │
       ▼           │
┌──────────────┐   │
│ OnActorBegin │   │
│   Overlap    │   │
└──────┬───────┘   │
       │           │
       ▼           │
┌──────────────┐   │
│ Identificar  │   │
│ Equipo del   │   │
│ Jugador      │   │
└──────┬───────┘   │
       │           │
       ▼           │
┌──────────────┐   │
│ Incrementar  │   │
│ RedCount o   │   │
│ BlueCount    │   │
│ en GM_Fps    │   │
└──────┬───────┘   │
       │           │
       ▼           │
┌──────────────┐   │
│UpdatePoints  │   │
│  OnFps()     │   │
└──────┬───────┘   │
       │           │
       ▼           │
┌──────────────┐   │
│  Respawn o   │───┘
│  Destroy     │
└──────────────┘
```

---

### Proyectiles: BP_CactusProjectileBlue / BP_CactusProjectileRed | **Clase Padre** | `Actor` |

**EventGraph (4 eventos cada uno):**
- `BeginPlay` - Inicialización del proyectil
- `OnHit` - Detección de colisión
- `Tick` - Actualización de movimiento
- Evento adicional para efectos/destrucción

La diferencia entre Blue y Red es puramente visual y de identificación de equipo.

---

#### WBP_MainHUD | **Clase Padre** | `UserWidget` |


**Funciones de Binding (para UI dinámica):**
| Función | Retorno | Propósito |
|---------|---------|-----------|
| `BlueCount` | `FText` | Texto con puntos azules |
| `RedCount` | `FText` | Texto con puntos rojos |
| `GetBluePercent` | `float` | Porcentaje (0-1) para barra de progreso azul |
| `GetRedPercent` | `float` | Porcentaje (0-1) para barra de progreso roja |

**Flujo de Actualización UI:**
```
GM_Fps.RedCount / BlueCount cambia
              │
              ▼
    UpdatePointsOnFps()
              │
              ▼
    BP_Fps.RedCount / BlueCount actualizados
              │
              ▼
    WBP_MainHUD detecta cambio (Binding)
              │
    ┌─────────┴─────────┐
    ▼                   ▼
RedCount()          BlueCount()
GetRedPercent()     GetBluePercent()
    │                   │
    ▼                   ▼
┌─────────┐       ┌─────────┐
│ Texto   │       │ Texto   │
│ Barra % │       │ Barra % │
└─────────┘       └─────────┘
```

#### WBP_PauseMenu | **Clase Padre** | `UserWidget` |

Maneja 4 eventos de botón:
1. Resume Game
2. Options
3. Return to Menu
4. Quit Game

#### WBP_WinEnd / WBP_LoseEnd | **Clase Padre** | `UserWidget` |

Pantallas simples de fin de partida con opciones básicas.

---

### Enhanced Input

**Mappings Definidos:**
| Acción | Teclado |
|--------|---------|
| Move | A/D/W/S |
| Look | Mouse XY |
| Menú | Esc |
| Shoot | Mouse Left Button|

### Sistema de Puntuación

```
┌─────────────┐
│   BP_Mine   │
│ (Overlap)   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  GM_Fps     │
│ RedCount++  │
│     o       │
│ BlueCount++ │
└──────┬──────┘
       │
       ▼
┌───────────────────┐
│ UpdatePointsOnFps │
└─────────┬─────────┘
          │
    ┌─────┴─────┐
    ▼           ▼
┌───────┐   ┌───────┐
│BP_Fps │   │BP_Fps │
│ (Red) │   │(Blue) │
│Update │   │Update │
└───────┘   └───────┘
          │
          ▼
┌─────────────────┐
│  WBP_MainHUD    │
│ Binding Update  │
│ (RedCount,      │
│  BlueCount,     │
│  Percentages)   │
└─────────────────┘
```

## Configuración-técnica

### Configuración de renderizado

**Archivo:** `Config/DefaultEngine.ini`

```ini
[/Script/Engine.RendererSettings]
r.AllowStaticLighting=False              ; Solo iluminación dinámica
r.GenerateMeshDistanceFields=True        ; Para Lumen/DFAO
r.DynamicGlobalIlluminationMethod=2      ; Lumen
r.ReflectionMethod=2                     ; Lumen Reflections
r.RayTracing=True                        ; Ray Tracing habilitado
r.Shadow.Virtual.Enable=1                ; Virtual Shadow Maps
r.Lumen.Supported.SM5=1                  ; Lumen en SM5
```

### Plataforma objetivo

| Windows |

---

## Inicio Rápido

### Requisitos
1. Unreal Engine 5.7
2. Steam instalado y ejecutándose

### Ejecutar el Proyecto

1. Abrir `HexMine.uproject` con UE 5.7
2. Esperar compilación de shaders
3. Presionar Play en el Editor

### Probar Multijugador Local

1. **Play Settings** → **Net Mode**: Play As Listen Server
2. **Number of Players**: 2+
3. Presionar Play
4. El primer jugador será el Host automáticamente

### Probar Multijugador Online (Steam)

**Prerequisitos:**
- Steam ejecutándose en ambos PC
- Ambos jugadores deben ser amigos en Steam

**Host:**
1. Ejecutar el juego empaquetado
2. Crear sesión via menú

**Cliente:**
1. Ejecutar el juego
2. Unirse via invitación de Steam o menú:

---

## Resumen de Blueprints

| Blueprint | Clase Padre | Propósito Principal |
|-----------|-------------|---------------------|
| BP_SteamSessionInstance | AdvancedFriendsGameInstance | Gestión de sesiones Steam |
| GM_Fps | GameModeBase | Game Mode principal (puntuación, spawn) |
| BP_Fps | Character | Personaje jugador con replicación |
| BP_Mine | Actor | Mina capturable para puntos |
| WBP_MainHUD | UserWidget | HUD con puntuaciones |
| WBP_PauseMenu | UserWidget | Menú de pausa |
| BP_FirstPersonPlayerController | PlayerController | Controlador del personaje |
| BP_CactusProjectileBlue | Actor | Proyectil equipo azul |
| BP_CactusProjectileRed | Actor | Proyectil equipo rojo |

---
