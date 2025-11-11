# xComfortd-Go: Datenstruktur-Diagramm

## Überblick der Hauptkomponenten

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Interface                                    │
│  (Zentrale Verwaltung aller Geräte & Datenpunkte)                   │
├─────────────────────────────────────────────────────────────────────┤
│ • datapoints: map[byte]*Datapoint    [DP Nummer → DP]               │
│ • devices: map[int]*Device           [Seriennummer → Device]        │
│ • txCommandChan: chan request        [TX-Befehle]                   │
│ • configCommandChan: chan request    [Config-Befehle]               │
│ • extendedCommandChan: chan request  [Extended Commands]            │
│ • handler: Handler                   [Callback-Handler]             │
│ • setupChan: chan datapoints         [DP-Setup-Kanal]               │
│ • txSemaphore: *semaphore.Weighted  [TX-Semaphore]                  │
│ • verbose: bool                      [Debug-Modus]                  │
└─────────────────────────────────────────────────────────────────────┘
         │
         ├─────────────────────────────────┬──────────────────────────────
         │                                 │
         ▼                                 ▼
    ┌─────────────────────┐        ┌──────────────────────────┐
    │      Device         │        │     Datapoint            │
    ├─────────────────────┤        ├──────────────────────────┤
    │ • deviceType        │        │ • device: *Device        │
    │ • subtype: byte     │        │ • name: string           │
    │ • serialNumber: int │        │ • number: byte           │
    │ • name: string      │        │ • channel: int           │
    │ • rssi: ...         │        │ • mode: int              │
    │ • battery: ...      │        │ • sensor: bool           │
    │ • datapoints: []DP  │        │ • queue: Queue           │
    └─────────────────────┘        └──────────────────────────┘
         │                                 │
         │                                 └─→ (gehört zu einem Device)
         │
         ▼ (hat verschiedene Typen)
    ┌──────────────────────────────────────────────┐
    │         Device Types (DeviceType)            │
    ├──────────────────────────────────────────────┤
    │ • DT_CSAx_01           (Switching Actuator)  │
    │ • DT_CDAx_01           (Dimming Actuator)    │
    │ • DT_CHAX_010x         (Heating Actuator)    │
    │ • DT_CJAU_0101/02/04   (Shutter)             │
    │ • DT_CTAA_01/02/04     (Battery Sensors)     │
    │ • DT_CRCA_...          (Remote Controls)     │
    │ • DT_CBMA_02           (Motion Detector)     │
    │ • DT_CHVZ_01           (HRV - Lüftung)       │
    └──────────────────────────────────────────────┘
```

## Datenfluss und Message-Struktur

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        RX Message (Eingang)                              │
│                     (von xComfort-Gerät empfangen)                       │
├─────────────────────────────────────────────────────────────────────────┤
│ Byte 0: Datenpunkt-Nummer (DP Number)                                    │
│ Byte 1: Event-Typ (RX_EVENT_*)                                           │
│         - RX_EVENT_STATUS (0x40)    → Status-Update                      │
│         - RX_EVENT_ON (0x50)        → Ein-Befehl                         │
│         - RX_EVENT_OFF (0x51)       → Aus-Befehl                         │
│         - RX_EVENT_VALUE (0x62)     → Wert-Update                        │
│         - RX_EVENT_STATUS_EXT       → Extended Status                    │
│         - Weitere Events...                                              │
│ Byte 2: Event-spezifische Daten                                          │
│ ...                                                                      │
│ Byte 7: Signal Strength (RSSI)                                           │
│ Byte 8: Battery State (lower 5 bits) + Cyclic-Flag (bit 5)               │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼ (wird verarbeitet in Interface.rx())
    ┌──────────────────────────────┐
    │  Datapoint.rx(Handler, data) │
    └──────────────────────────────┘
         │
         ├──→ dp.status(handler, statusByte)
         │    ├─ IsSwitchingActuator() → StatusBool(on/off)
         │    ├─ IsDimmingActuator()   → StatusValue(0-255)
         │    └─ IsShutter()           → shutterStatus(...)
         │
         └──→ dp.event(handler, event, data)
              └─ Verschiedene Datentypen:
                 • RX_DATA_TYPE_UINT8      (1 Byte)
                 • RX_DATA_TYPE_UINT16     (2 Bytes, Big-Endian)
                 • RX_DATA_TYPE_INT16_1POINT (2 Bytes / 10)
                 • RX_DATA_TYPE_UINT16_2POINT (2 Bytes / 100)
                 • RX_DATA_TYPE_UINT32     (4 Bytes, Big-Endian)
                 • RX_DATA_TYPE_FLOAT      (4 Bytes, IEEE 754)
                 • RX_DATA_TYPE_PERCENT    (als Prozent 0-100%)
                 • RX_DATA_TYPE_RC_DATA    (Remote Control-Daten)
```

## Status und Zustandsverwaltung

### Battery State (BatteryState)
```
┌─────────────────────────────────┐
│      BatteryState (byte)        │
├─────────────────────────────────┤
│ 0   → N/A          (0%)         │
│ 1   → empty        (20%)        │
│ 2   → very weak    (40%)        │
│ 3   → weak         (60%)        │
│ 4   → good         (80%)        │
│ 5   → new          (100%)       │
│ 16  → mains-powered             │
│ 17+ → error                     │
└─────────────────────────────────┘
```

### Signal Strength (RSSI)
```
┌──────────────────────────────────┐
│      SignalStrength (byte)       │
├──────────────────────────────────┤
│ ≤67   → good                     │
│ ≤75   → normal                   │
│ ≤90   → weak                     │
│ ≤120  → very weak                │
│ >120  → error                    │
└──────────────────────────────────┘
```

## Handler - Callback-Interface

```
┌───────────────────────────────────────────────────────────────────┐
│                      Handler Interface                             │
│     (Empfängt Callbacks bei Datenänderungen)                       │
├───────────────────────────────────────────────────────────────────┤
│ StatusValue(dp *Datapoint, value int)                              │
│   → Numerischer Wert (z.B. Dimmer 0-255)                           │
│                                                                   │
│ StatusBool(dp *Datapoint, on bool)                                 │
│   → Boolean Status (an/aus)                                        │
│                                                                   │
│ StatusShutter(dp *Datapoint, status ShutterStatus)                │
│   → Jalousien-Status                                              │
│                                                                   │
│ Event(dp *Datapoint, event Event)                                  │
│   → Einfache Events (On, Off, UpPressed, etc.)                     │
│                                                                   │
│ Wheel(dp *Datapoint, value interface{})                            │
│   → RC Wheel-Position (Remote Control)                             │
│                                                                   │
│ Value(dp *Datapoint, value interface{})                            │
│   → Generischer Wert                                              │
│                                                                   │
│ ValueEvent(dp *Datapoint, event Event, value interface{})         │
│   → Event mit begleitendem Wert                                    │
│                                                                   │
│ Battery(device *Device, percentage int)                            │
│   → Batterie-Prozentsatz (0-100%)                                  │
│                                                                   │
│ Rssi(device *Device, rssi int)                                     │
│   → Signal-Stärke                                                 │
│                                                                   │
│ Power(device *Device, value interface{})                           │
│   → Stromverbrauch (für unterstützte Geräte)                       │
│                                                                   │
│ InternalTemperature(device *Device, centigrade int)                │
│   → Interne Temperatur                                            │
│                                                                   │
│ DPLChanged()                                                       │
│   → Datenpunkt-Liste hat sich geändert                             │
└───────────────────────────────────────────────────────────────────┘
```

## Event-Typen

### Empfangene Events (RX Events)
```
┌──────────────────────────────────────────┐
│          Receiver Events                  │
├──────────────────────────────────────────┤
│ EventOn/EventOff                          │
│ EventSwitchOn/EventSwitchOff              │
│ EventUpPressed/EventUpReleased            │
│ EventDownPressed/EventDownReleased        │
│ EventForced                               │
│ EventSingleOn                             │
│ EventValue                                │
│ EventTooCold/EventTooWarm                 │
└──────────────────────────────────────────┘
```

### Gesendete Events (TX Events)
```
┌────────────────────────────────────────────┐
│          Transmit Events/Commands           │
├────────────────────────────────────────────┤
│ SWITCH:    MCI_TED_OFF / MCI_TED_ON        │
│ DIM:       MCI_TED_DARKER / BRIGHTER       │
│ JALOUSIE:  MCI_TED_OPEN / CLOSE / STOP     │
│ PUSHBUTTON: MCI_TED_UP_PRESSED/RELEASED    │
│ BASICMODE: MCI_TED_LEARNMODE_*             │
│ DIRECT:    MCI_TED_DIRECT_ON/OFF           │
└────────────────────────────────────────────┘
```

## Device-Hierarchie und Funktionen

```
┌──────────────────────────────────────────────────────────────┐
│                    Device (generisch)                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ├─→ IsSwitchingActuator()  → Schalter/Relais              │
│  │   • CSAx_01, CSAU_0101, CBEU_0201                       │
│  │   ├─→ StatusBool(on/off)                                │
│  │   └─→ TX: ON/OFF Commands                               │
│  │                                                          │
│  ├─→ IsDimmingActuator()    → Dimmer                        │
│  │   • CDAx_01, CDAx_01NG, CAAE_01                         │
│  │   ├─→ StatusValue(0-255)                                │
│  │   ├─→ ReportsPower() → Stromverbrauch                   │
│  │   └─→ TX: DIM Commands                                  │
│  │                                                          │
│  ├─→ IsHeatingActuator()    → Heizregler                   │
│  │   • CHAX_010x                                            │
│  │   ├─→ ReportsPower()                                    │
│  │   └─→ Extended Status für Steuerung                     │
│  │                                                          │
│  ├─→ IsShutter()            → Jalousien/Rollläden          │
│  │   • CJAU_0101, 0102, 0104                               │
│  │   ├─→ StatusShutter(status)                             │
│  │   └─→ TX: OPEN/CLOSE/STOP Commands                      │
│  │                                                          │
│  └─→ IsBatteryOperated()    → Batterie-betrieb             │
│      • CTAA_*, CRCA_*, CTEU_02, CBMA_02, etc.              │
│      ├─→ Battery-State Tracking                            │
│      └─→ RSSI Monitoring                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Befehlswarteschlangen (Command Queues)

```
┌─────────────────────────────────────────┐
│  TX Command Queue (txCommandChan)       │
│  Semaphore-gesteuert                    │
├─────────────────────────────────────────┤
│ Befehle von Handler → Device            │
│ Gewichtet/Serialisiert mit Semaphore    │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Config Command Queue                   │
│  Mutex-geschützt                        │
├─────────────────────────────────────────┤
│ Konfiguration & Lernmodus-Befehle       │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Extended Command Queue                 │
│  Mutex-geschützt                        │
├─────────────────────────────────────────┤
│ Extended Status & Erweiterte Befehle    │
└─────────────────────────────────────────┘
```

## Nachrichtentypen (Message Packet Types - MCI_PT_*)

```
┌──────────────────────────────────────────────────────┐
│         Nachrichtentypen im xComfort-Protokoll      │
├──────────────────────────────────────────────────────┤
│ 0xB1  MCI_PT_TX           → Senden an Gerät        │
│ 0xB2  MCI_PT_CONFIG       → Konfiguration          │
│ 0xC1  MCI_PT_RX           → Empfangen von Gerät    │
│ 0xC3  MCI_PT_STATUS       → Status-Abfrage         │
│ 0xD1  MCI_PT_EXTENDED     → Extended Status/Befehle│
└──────────────────────────────────────────────────────┘
```

## Zusammenfassung: Datenfluss

### RX-Richtung (Empfang von xComfort → USB → MQTT)

```
┌─────────────────┐
│  xComfort-Netz  │
└────────┬────────┘
         │
         ▼ (empfangen)
    ┌─────────────────┐
    │  USB/HID-Layer  │
    └────────┬────────┘
             │
             ▼
        ┌──────────────────┐
        │ Interface.rx()   │
        └────────┬─────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
   ┌─────────────┐  ┌──────────────────┐
   │ Datapoint   │  │ Extended Status  │
   │ .rx()       │  │ Verarbeitung     │
   └────────┬────┘  └────────┬─────────┘
            │                │
            ▼                ▼
       ┌──────────────────────────┐
       │  Handler Callbacks       │
       │  (StatusValue, Wheel...)  │
       └──────────┬───────────────┘
                  │
                  ▼
       ┌──────────────────┐
       │ MQTT-Publishing  │
       │  (retained=true) │
       └────────┬─────────┘
                │
                ▼
       ┌──────────────────┐
       │ MQTT Broker      │
       │ Home Assistant   │
       └──────────────────┘
```

---

# MQTT-zu-USB Datenfluss (TX-Richtung)

## TX-Richtung (Befehl von MQTT → USB → xComfort)

```
┌──────────────────────────────────────────────────────────────────┐
│                    Home Assistant / MQTT Client                   │
│              (publiziert Befehl zu MQTT-Topic)                    │
└────────────────────────┬─────────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
    ┌─────────────┐               ┌──────────────┐
    │ /set/switch │               │ /set/dimmer  │
    │ (true/false)│               │ (0-255)      │
    └─────┬───────┘               └──────┬───────┘
          │                              │
          │ (Topic-Pattern: clientId/DP/set/...)
          │
          ▼
    ┌──────────────────────────────────────────────┐
    │      MqttRelay.switchCallback()              │
    │      MqttRelay.dimmerCallback()              │
    │      MqttRelay.shutterCallback()             │
    │      ...andere Callbacks...                   │
    └────────────────┬─────────────────────────────┘
                     │
                     ▼ (Befehl dekodieren)
            ┌────────────────────────────┐
            │  Datapoint gefunden?       │
            │  dp := r.Datapoint(dpNum)  │
            └────────────┬───────────────┘
                         │
                    ┌────┴────┐
                    │          │
                Yes ▼          ▼ No
           ┌────────────┐  (Fehler)
           │  ctx       │  unknown
           │  gegeben   │  datapoint
           └────┬───────┘
                │
                ▼
    ┌──────────────────────────────────────────┐
    │  Datapoint.Switch(ctx, on bool)          │
    │  Datapoint.Dim(ctx, value int)           │
    │  Datapoint.Shutter(ctx, command)         │
    │  Datapoint.DesiredTemperature(ctx, val)  │
    │  ...andere Befehle...                     │
    └────────────┬─────────────────────────────┘
                 │
                 ▼ (TX-Befehl zusammenstellen)
    ┌──────────────────────────────────────────┐
    │  d.queue.Lock()  (Queue-Management)      │
    │  Befehl in Bytes: [DP#, MCI_TE_*, Daten] │
    │  d.queue.Unlock()                        │
    └────────────┬─────────────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────┐
    │ iface.sendTxCommand(ctx, []byte{...})    │
    │ • Semaphore-kontrolliert (txSemaphore)   │
    │ • Kanal: txCommandChan                   │
    │ • Wartet auf TX-Verarbeitung             │
    └────────────┬─────────────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────┐
    │   TX-Queue Verarbeiter (Main Loop)       │
    │   • Liest aus txCommandChan              │
    │   • Sendet über USB Write()              │
    └────────────┬─────────────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────┐
    │      USB Device Output Endpoint 2        │
    │     (usb.out.WriteContext())             │
    └────────────┬─────────────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────┐
    │    Broadcast an xComfort-Netz            │
    │    (RF 868 MHz)                          │
    └──────────────────────────────────────────┘
```

---

## Detaillierter MQTT-Callback-Fluss

```
┌─────────────────────────────────────────────────────────────────┐
│                    MQTT Topic Abonnierung                        │
├─────────────────────────────────────────────────────────────────┤
│ Topic-Pattern: "xcomfort/NN/set/*"  (NN = Datenpunkt-Nummer)    │
│                                                                 │
│ Registrierte Callbacks:                                         │
│ • xcomfort/NN/set/switch         → switchCallback()            │
│ • xcomfort/NN/set/dimmer         → dimmerCallback()            │
│ • xcomfort/NN/set/shutter        → shutterCallback()           │
│ • xcomfort/NN/set/temperature    → desiredTemperatureCallback()│
│ • xcomfort/NN/set/current        → currentTemperatureCallback()│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## USB-Kommunikation Detail

### USB Device Endpoints

```
┌──────────────────────────────────┐
│    xComfort USB Gerät            │
│    VID: 0x188a, PID: 0x1101      │
├──────────────────────────────────┤
│                                  │
│  Endpoint 1 (IN - Empfang)       │
│  ├─ Datenrate: Max Packet Size   │
│  └─ ReadContext() → rx-Messages  │
│                                  │
│  Endpoint 2 (OUT - Senden)       │
│  ├─ Datenrate: Max Packet Size   │
│  └─ WriteContext() → tx-Commands │
│                                  │
└──────────────────────────────────┘
```

### USB Read/Write Wrapper (usbDevice struct)

```
┌────────────────────────────────────────────┐
│        usbDevice (io.ReadWriteCloser)      │
├────────────────────────────────────────────┤
│ • ctx: context.Context                     │
│ • in: *gousb.InEndpoint                    │
│ • out: *gousb.OutEndpoint                  │
│ • dev: *gousb.Device                       │
│ • done: cleanup callback                   │
├────────────────────────────────────────────┤
│ Read(p []byte)  → in.ReadContext()         │
│ Write(p []byte) → out.WriteContext()       │
│ Close()         → cleanup                  │
└────────────────────────────────────────────┘
```

---

## Command Queue Management

### TX Command Flow

```
┌──────────────────────────────────┐
│   Datapoint.Switch/Dim/etc.      │
│   aufgerufen von MQTT-Callback   │
└────────────┬─────────────────────┘
             │
             ▼ d.queue.Lock()
        ┌─────────────────────┐
        │ Check: Ist dies das │
        │ neueste Kommando?   │
        │ (!last → verwerfen) │
        └────────┬────────────┘
                 │
                 ▼ if !last: return nil (abbrechen)
                 │
                 ▼
    ┌───────────────────────────────────────┐
    │ iface.sendTxCommand(ctx, bytes{...})  │
    └────────────┬────────────────────────┘
                 │
                 ▼ txSemaphore.Acquire(ctx)
            ┌─────────────────────────────┐
            │ Warte auf Semaphore         │
            │ (Begrenzt gleichzeitige TX) │
            └────────┬────────────────────┘
                     │
                     ▼ Befehl in txCommandChan
                ┌────────────────────────┐
                │  txCommandChan   ◄── │
                │  (request-Struktur)   │
                └────────┬───────────────┘
                         │
                         ▼ (Main Loop verarbeitet)
                    ┌──────────────┐
                    │ USB.Write()  │
                    └──────┬───────┘
                           │
                           ▼ txSemaphore.Release()
                      ┌──────────────┐
                      │ Freigeben    │
                      └──────────────┘
```

---

## MqttRelay Struktur

```
┌────────────────────────────────────────────┐
│         MqttRelay (embeddet Interface)     │
├────────────────────────────────────────────┤
│ • client: mqtt.Client    (MQTT-Verbindung)│
│ • ctx: context.Context   (Context-Control)│
│ • clientId: string       (MQTT Client-ID) │
│ • haDiscoveryPrefix: string                │
│ • haDiscoveryAutoremove: bool              │
│                                            │
│ Embedded: xc.Interface                     │
│  ├─ datapoints: map[byte]*Datapoint        │
│  ├─ devices: map[int]*Device               │
│  └─ Handler callbacks                      │
├────────────────────────────────────────────┤
│ Handler Callbacks:                         │
│ • StatusBool(dp, bool)                     │
│ • StatusValue(dp, int)                     │
│ • StatusShutter(dp, status)                │
│ • Event(dp, event)                         │
│ • Battery(dev, percent)                    │
│ • Rssi(dev, rssi)                          │
│ • Power(dev, value)                        │
│ • InternalTemperature(dev, temp)           │
│                                            │
│ MQTT Publish:                              │
│ • publish(topic, retained, payload)        │
└────────────────────────────────────────────┘
```

---

## MQTT Topic Struktur

```
┌─────────────────────────────────────────────────┐
│         MQTT Topic Namensräume                  │
├─────────────────────────────────────────────────┤
│                                                 │
│ GET (Empfänger publiziert):                    │
│ • {clientId}/NN/get/switch    → true/false    │
│ • {clientId}/NN/get/dimmer    → 0-255         │
│ • {clientId}/NN/get/shutter   → open/close/.. │
│ • {clientId}/NN/get/value     → float/int     │
│ • {clientId}/NN/event         → Ereignisse    │
│ • {clientId}/NN/wheel         → RC Pos.       │
│ • {clientId}/NN/valve         → Ventil Pos.   │
│ • {clientId}/NN/battery       → Prozent       │
│ • {clientId}/NN/rssi          → Signal Stärke │
│ • {clientId}/NN/power         → Watts         │
│ • {clientId}/NN/internal_temp → °C            │
│                                                 │
│ SET (Client abonniert):                        │
│ • {clientId}/NN/set/switch    ← true/false    │
│ • {clientId}/NN/set/dimmer    ← 0-255         │
│ • {clientId}/NN/set/shutter   ← open/close/.. │
│ • {clientId}/NN/set/temperature ← float       │
│ • {clientId}/NN/set/current   ← float         │
│                                                 │
│ HA Discovery Topics:                           │
│ • homeassistant/light/xc_NN/config            │
│ • homeassistant/switch/xc_NN/config           │
│ • homeassistant/cover/xc_NN/config            │
│ • etc.                                         │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Zusammenfassungs-Architektur

```
┌──────────────────────────────────────────────────────────────────┐
│                        Home Assistant / MQTT                      │
└──────────────────────────┬───────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
         ┌─────────────┐          ┌──────────────┐
         │ MQTT Publish│          │ MQTT Subscribe
         │  (Status)   │          │   (Befehle)  │
         └─────┬───────┘          └──────┬───────┘
               │                        │
               │                        ▼
               │                   ┌────────────────┐
               │                   │ MqttRelay      │
               │                   │ .connected()   │
               │                   │ subscribe()    │
               │                   └────────┬───────┘
               │                            │
               │                  ┌─────────┴──────────┐
               │                  │                    │
               │                  ▼                    ▼
               │            ┌──────────────┐    ┌──────────────┐
               │            │ switchCallback    │ dimmerCallback
               │            └──────┬────────┘    └──────┬───────┘
               │                   │                    │
               │                   ▼                    ▼
               │            ┌────────────────────────────────┐
               │            │ xc.Datapoint.Switch/Dim/etc   │
               │            │ → sendTxCommand()              │
               │            └────────┬─────────────────────┘
               │                     │
               │                     ▼
               │            ┌────────────────────┐
               │            │ txCommandChan      │
               │            │ (USB-Sende-Queue)  │
               │            └────────┬───────────┘
               │                     │
               │                     ▼
               │            ┌────────────────────┐
               │            │ USB Write()        │
               │            │ → xComfort-Netz   │
               │            └────────┬───────────┘
               │                     │
               └─────────────────────┼──────────────────┐
                                     │                  │
                                     ▼                  │
                            ┌────────────────┐         │
                            │ xComfort Netz  │         │
                            │ (868 MHz RF)   │         │
                            └────────┬───────┘         │
                                     │                 │
                                     ▼                 │
                            ┌────────────────────┐     │
                            │ Actuator empfängt  │     │
                            │ Befehl und handelt │     │
                            │ darauf (ON/OFF...)│      │
                            └────────┬───────────┘     │
                                     │                 │
                                     ▼                 │
                            ┌─────────────────────┐    │
                            │ Status zurück zum   │    │
                            │ Netz senden         │    │
                            └────────┬────────────┘    │
                                     │                 │
                                     ▼                 │
                            ┌─────────────────────┐    │
                            │ USB Read()          │◄───┘
                            │ Interface.rx()      │
                            └────────┬────────────┘
                                     │
                                     ▼
                            ┌─────────────────────┐
                            │ Datapoint.rx()      │
                            │ Handler Callbacks   │
                            └────────┬────────────┘
                                     │
                                     ▼
                            ┌─────────────────────┐
                            │ MqttRelay.publish() │
                            │ → MQTT-Broker       │
                            └────────┬────────────┘
                                     │
                                     ▼
                            ┌─────────────────────┐
                            │ Home Assistant      │
                            │ Status aktualisiert │
                            └─────────────────────┘
```

## Datentyp-Mapping in RX Events

```
┌────────────────────────────────────┐
│   Datentyp → Verarbeitung         │
├────────────────────────────────────┤
│ UINT8           → Byte direkt     │
│ UINT16          → 2 Bytes (BE)    │
│ INT16_1POINT    → / 10            │
│ UINT16_2POINT   → / 100           │
│ UINT16_3POINT   → / 1000          │
│ UINT32          → 4 Bytes (BE)    │
│ UINT32_3POINT   → / 1000          │
│ FLOAT           → IEEE 754        │
│ PERCENT         → * 100 / 255     │
│ RC_DATA         → Position + Wheel│
│ RCT_OUT         → Feuchte + Temp  │
└────────────────────────────────────┘
```

