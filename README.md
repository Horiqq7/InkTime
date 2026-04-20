# InkTime v6 — Open Hardware Smartwatch cu E-Paper

InkTime v6 este un smartwatch open-source bazat pe **nRF52840**, optimizat pentru consum redus și autonomie mare, integrat într-o carcasă compactă.

---

## Cuprins

- [Diagramă bloc](#diagramă-bloc)
- [Bill of Materials (BOM)](#bill-of-materials-bom)
- [Funcționalitate hardware detaliată](#funcționalitate-hardware-detaliată)
- [Maparea pinilor nRF52840](#maparea-pinilor-nrf52840)
- [Analiza consumului de energie](#analiza-consumului-de-energie)
- [Design PCB și integrare mecanică](#design-pcb-și-integrare-mecanică)
- [Design log și observații pentru review](#design-log-și-observații-pentru-review)
- [Structura repository-ului](#structura-repository-ului)
- [Licență](#licență)

---

## Diagramă bloc

```
  ┌──────────┐  VBUS   ┌────────────┐  VBAT  ┌───────────┐  3V3
  │  USB-C   ├────────►│  BQ25180   ├───────►│  RT6160A  ├──────► Sistem
  │   (J4)   │         │ (Charger)  │        │ (Buck DC) │
  └──────────┘         └─────┬──────┘        └─────┬─────┘
       │ ESD                 │ VBAT                 │ EN
       ▼                     ▼                      ▼
  ┌──────────┐         ┌──────────┐          ┌──────────┐
  │USBLC6-2  │         │  LiPo    │          │ DMG2305  │
  │  (ESD)   │         │ Baterie  │          │  (PMOS)  │
  └──────────┘         └────┬─────┘          └──────────┘
                            │ I2C
                            ▼
                      ┌──────────┐
                      │ MAX17048 │
                      │  (Fuel   │
                      │  Gauge)  │
                      └──────────┘

                   ┌──────────────────────────────┐
                   │          nRF52840             │
                   │  ARM M4F │ BLE 5.x │ USB FS  │
                   └──┬───────────┬───────────┬───┘
                      │ SPI       │ I2C       │ GPIO / SWD
                      ▼           ▼           ▼
               ┌────────────┐  ┌───────────────────────┐  ┌──────────┐
               │ E-Paper    │  │ BMA423 │MAX17048│DRV2605│  │ TC2030  │
               │ 1.54" (J1) │  │  IMU   │ Fuel G.│Haptic │  │ (Debug) │
               └────────────┘  └───────────────────────┘  └──────────┘
                                                  │ N-MOS
                                            ┌─────▼──────┐
                                            │ SI1308EDL  │
                                            │Motor haptic│
                                            └────────────┘
                                    RF
                   nRF52840 ───────────► 2450AT18B100E (Antenă)
```

**Flux de funcționare:**
- USB-C alimentează `BQ25180` (charger) și bateria LiPo.
- `RT6160A` generează 3.3V stabil; comutat prin PMOS `DMG2305UX`.
- `nRF52840` colectează date de la IMU / fuel gauge (I2C) și controlează display-ul (SPI).
- `DRV2605` generează feedback haptic, comutând motorul prin `SI1308EDL`.
- BLE asigură schimbul de date cu telefonul.

---

## Bill of Materials (BOM)

| # | Ref | Componentă | Cod / Valoare |
|---|-----|------------|---------------|
| 1 | U1 | MCU BLE | nRF52840-QIAA |
| 2 | IC2 | Li-Ion Charger | BQ25180YBGR |
| 3 | IC9 | Buck DC/DC | RT6160AWSC |
| 4 | U3 | Fuel Gauge | MAX17048G+T10 |
| 5 | IC3 | IMU | BMA423 |
| 6 | IC1 | Driver haptic | DRV2605YZFR |
| 7 | D1 | Protecție ESD USB | USBLC6-2SC6Y |
| 8 | Q1 | PMOS load switch | DMG2305UX-7 |
| 9 | Q3 | N-MOS (motor) | SI1308EDL-T1-GE3 |
| 10 | J1 | Conector FPC EPD | 503480-2400 (24p) |
| 11 | J4 | Conector USB-C | KH-TYPE-C-16P |
| 12 | ANT1 | Antenă chip 2.4GHz | 2450AT18B100E |
| 13 | J2 | Header debug | TC2030-IDC |
| 14 | X1 | Cristal HF | 32MHz |
| 15 | X2 | Cristal LF | 32.768kHz |

---

## Funcționalitate hardware detaliată

**1) nRF52840 — procesor și conectivitate**
ARM Cortex-M4F @ 64MHz, 1MB Flash, 256KB RAM, BLE 5.x. Rulează logica aplicației, controlează display-ul pe SPI, centralizează senzorii pe I2C și gestionează notificările BLE și feedback-ul haptic.

**2) Display e-paper 1.54"**
Comandat pe SPI cu semnale separate de control (DC, RST, BUSY, CS) prin conector FPC 24p. Consum aproape zero în stare statică; energia se consumă exclusiv la refresh.

**3) IMU — BMA423**
Conectat pe I2C. Folosit pentru step counting, wrist-raise detection și wake-on-motion. Poate genera întreruperi hardware pentru minimizarea polling-ului.

**4) Management baterie — BQ25180 + MAX17048 + RT6160A**
`BQ25180` încarcă bateria LiPo din VBUS. `MAX17048` estimează SOC și tensiunea bateriei. `RT6160A` convertește VBAT → 3.3V pentru întregul sistem. Lanțul permite operare corectă pe USB și pe baterie.

**5) Haptic — DRV2605 + motor**
`DRV2605` generează profile haptice via I2C + GPIO TRIG. Etajul de putere `SI1308EDL` comută motorul pentru feedback la notificări și interacțiuni UI.

**6) USB-C, ESD și debug**
`USBLC6-2` protejează liniile D+/D- împotriva ESD. Header-ul `TC2030` expune SWD pentru flash/debug și test de producție.

---

## Maparea pinilor nRF52840

| Pin nRF52840 | Semnal | Interfață | Direcție | Motiv alegere |
|-------------|--------|-----------|----------|---------------|
| P0.02 | EPD_SCK | SPI | Out | Clock display, grupat compact cu celelalte linii SPI |
| P0.03 | EPD_MOSI | SPI | Out | Date grafice spre e-paper |
| P0.28 | EPD_MISO | SPI | In | Citire status display |
| P0.29 | EPD_CS | GPIO/SPI | Out | Chip select e-paper |
| P0.04 | EPD_DC | GPIO | Out | Selectare mod Data / Comandă |
| P0.05 | EPD_RST | GPIO | Out | Reset hardware display |
| P0.06 | EPD_BUSY | GPIO | In | Sincronizare stare busy |
| P0.26 | I2C_SDA | I2C | I/O | Magistrală comună: BMA423, BQ25180, MAX17048, DRV2605 |
| P0.27 | I2C_SCL | I2C | Out | Clock I2C comun senzori/putere/haptic |
| P0.13 | SW_UP | GPIO IRQ | In | Buton sus, întrerupere la apăsare |
| P0.14 | SW_ENT | GPIO IRQ | In | Buton enter, wake-up din sleep |
| P0.15 | SW_DN | GPIO IRQ | In | Buton jos, navigare meniu |
| P0.18 | USB_D- | USB FS | I/O | Linie USB dedicată (pin impus de silicon) |
| P0.20 | USB_D+ | USB FS | I/O | Linie USB dedicată (pin impus de silicon) |
| P0.00 / P0.01 | XL1 / XL2 | LFCLK | I/O | Cristal 32.768kHz pentru RTC low-power |
| XC1 / XC2 | X1 32MHz | HFCLK | I/O | Ceas principal CPU/RF |
| SWDIO | SWD_DATA | SWD | I/O | Programare și debugging |
| SWDCLK | SWD_CLK | SWD | In | Clock SWD |
| SWO | SWO_TRACE | SWD | Out | Trace / telemetrie debug |
| ANT | Antenă BLE | RF | Out | Ieșire RF spre rețea de matching |

---

## Analiza consumului de energie

| Bloc | Curent activ | Curent sleep/idle | Duty cycle | Curent mediu |
|------|-------------|-------------------|------------|--------------|
| nRF52840 (CPU + BLE) | 3.2–4.8mA | 1.5µA | ~5–6% | ~210µA |
| E-Paper (refresh) | ~15mA | ~0µA static | ~0.5% | ~75µA |
| BMA423 | ~150µA | ~14µA | ~10% | ~28µA |
| MAX17048 | 23µA | 23µA | 100% | 23µA |
| DRV2605 + motor | ~50mA vârf | ~0.25µA | ~0.1% | ~50µA |
| RT6160A quiescent | — | ~25µA | 100% | 25µA |
| **Total estimat** | | | | **~411µA** |

Pentru baterie de 250mAh: autonomie teoretică **≈ 608h (~25 zile)**; cu sleep agresiv și refresh rar: **30–40+ zile**.

---

## Design PCB și integrare mecanică

Layout realizat conform fișierului cu dimensiuni mecanice (`inktime.fbrd`). Topologia include zona RF dedicată cu keepout de cupru și decupaj PCB sub antenă, secțiune de alimentare separată și trasee scurte spre conectorul EPD. Trasee de alimentare 0.3mm, semnale date 0.15mm minim. Planuri de masă pe TOP și BOTTOM cu via stitching în zona RF. Modele 3D disponibile în `Mechanical/InkTime3D.f3z` și `Mechanical/InkTime3D.step`.

---

## Design log și observații pentru review

**Design log (sintetic):**
1. Definire arhitectură (MCU, alimentare, senzori, display)
2. Captură schemă și validare ERC
3. Placement cu prioritizare RF / alimentare / display
4. Rutare + optimizări DRC + planuri de masă
5. Export producție și validare mecanică 3D

**Observații utile pentru review:**
- Verificați keepout-ul și decupajul PCB sub antenă `ANT1` conform recomandărilor vendor
- Confirmați polaritatea bateriei LiPo la test pad-uri
- Respectați secvența de power-up EPD din firmware pentru a evita refresh instabil
- Erorile DRC de tip `Dimension` (butoane, USB-C) sunt neglijate conform specificațiilor proiectului
- Test incremental recomandat: rail-uri → SWD → I2C → EPD → BLE

---

## Structura repository-ului

```
inktime/
├── Hardware/
│   ├── project-tsc.sch
│   ├── project-tsc-2d-pcb.brd
│   └── project-tsc.pdf
├── Manufacturing/
│   ├── gerbers.zip
│   ├── inktime.bom
│   └── inktime.cpl
├── Mechanical/
│   ├── InkTime3D.f3z
│   └── InkTime3D.step
├── Images/
├── LICENSE
└── README.md
```

---

## Licență

Proiectul este distribuit conform fișierului `LICENSE` din acest repository.
