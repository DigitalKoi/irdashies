# Rust ViewModel для Standings (аналіз TS моделей)

## 1. Джерела даних у TypeScript
Основні моделі та пропси, що використовуються Standings UI:

- `Standings`, `Gap`, `LastTimeState` з `src/frontend/components/Standings/createStandings.ts`.
- `DriverRowInfoProps` (фактичний набір даних для рендеру рядка) з `DriverInfoRow.tsx`.
- Форматування імен: `DriverNameParts`, `DriverNameFormat` з `DriverName.tsx`.
- Локальні пропси для елементів:
  - `DriverRatingBadgeProps` (badge: license + rating)
  - `RatingChangeProps` (iRating delta)
  - `CarManufacturerProps` (carId)
  - `CountryFlagProps` (flairId)
  - `CompoundProps` (tireCompound)
  - `DriverClassHeaderProps` (class name/color/sof/total)
  - `TitleBarProps`, `SessionBarProps` (header/footer дані)

Ці моделі побудовані на даних iRSDK (`src/app/irsdk/types`), але для UI використовуються лише вибрані поля.

---

## 2. Пропозиція Rust ViewModel структур (UI‑only)
Нижче — рекомендовані структури для ViewModel шару (тільки дані для UI, без бізнес‑логіки).

> **Принцип:** мінімізувати обсяг даних, і по можливості не виконувати дорогих конверсій типів при кожному кадрі.

```rust
/// Стан останнього кола
/// - SessionFastest: час дорівнює найкращому часу сесії
/// - PersonalBest: час дорівнює найкращому часу водія
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum LastTimeState {
    SessionFastest,
    PersonalBest,
}

/// Різниця від лідера класу
#[derive(Clone, Copy, Debug, Default)]
pub struct Gap {
    /// секунди, якщо відомо (Option::None = немає достатніх даних для розрахунку)
    pub value: Option<f32>,
    /// різниця в колах (0 = без різниці, >0 = позаду лідера)
    pub laps: i16,
}

/// Значення TrackSurface з iRSDK
#[repr(i8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum TrackSurface {
    NotInWorld = -1,
    OffTrack = 0,
    OnTrack = 1,
    PitLane = 2,
}

/// Дані про водія, які рідко змінюються
#[derive(Clone, Debug)]
pub struct DriverStaticVm {
    pub car_idx: i32,
    pub name: std::sync::Arc<str>,
    pub car_num: std::sync::Arc<str>,
    pub license: std::sync::Arc<str>,
    pub rating: i32,
    pub flair_id: Option<i32>,
    pub team_name: Option<std::sync::Arc<str>>,
    pub car_id: Option<i32>,
}

/// Дані про клас автомобіля
#[derive(Clone, Debug)]
pub struct CarClassVm {
    pub id: i32,
    pub color: u32, // RGB packed (як у TS)
    pub name: std::sync::Arc<str>,
    pub relative_speed: f32,
    pub est_lap_time: f32,
}

/// Прапори стану (можна як bitflags або прості bool)
#[derive(Clone, Copy, Debug, Default)]
pub struct DriverFlags {
    pub is_player: bool,
    pub on_pit_road: bool,
    pub on_track: bool,
    pub radio_active: bool,
    pub has_fastest_time: bool,
    pub dnf: bool,
    pub repair: bool,
    pub penalty: bool,
    pub slowdown: bool,
}

/// Динамічні (пер-кадрові) дані рядка
#[derive(Clone, Debug)]
pub struct DriverRowVm {
    pub car_idx: i32,
    pub class_position: Option<i32>,
    pub position: Option<i32>,
    /// Різниця до лідера (секунди) в межах сесії
    pub delta: Option<f32>,
    /// Різниця до лідера КЛАСУ (секунди/кола)
    pub gap: Option<Gap>,
    /// Інтервал до попередньої машини (секунди)
    pub interval: Option<f32>,
    pub last_time: Option<f32>,
    pub last_time_state: Option<LastTimeState>,
    pub fastest_time: Option<f32>,
    pub tire_compound: Option<i32>,
    pub lap_time_deltas: Option<Vec<f32>>,
    pub last_pit_lap: Option<i32>,
    pub last_lap: Option<i32>,
    /// Поточний і попередній TrackSurface (enum замість магічних чисел)
    pub prev_car_track_surface: Option<TrackSurface>,
    pub car_track_surface: Option<TrackSurface>,
    pub current_session_type: Option<std::sync::Arc<str>>,
    pub irating_change: Option<i32>,
    pub flags: DriverFlags,
    pub driver: DriverStaticVm,
    pub car_class: CarClassVm,
}

/// Група по класу
#[derive(Clone, Debug)]
pub struct StandingsClassVm {
    pub class_id: i32,
    pub class_name: std::sync::Arc<str>,
    pub class_color: u32,
    pub total_drivers: Option<i32>,
    pub sof: Option<i32>,
    pub rows: Vec<DriverRowVm>,
}

/// Повна модель виджета Standings
#[derive(Clone, Debug)]
pub struct StandingsVm {
    pub classes: Vec<StandingsClassVm>,
    pub is_multi_class: bool,
    pub hide_car_manufacturer: bool,
    pub highlight_color: u32,
}
```

> Якщо потрібне форматування імен (TS: `DriverNameParts`, `DriverNameFormat`), у Rust можна зберігати частини імені окремо:

```rust
#[derive(Clone, Debug)]
pub struct DriverNamePartsVm {
    pub first_name: std::sync::Arc<str>,
    pub middle_name: Option<std::sync::Arc<str>>,
    pub surname: std::sync::Arc<str>,
}

#[derive(Clone, Copy, Debug)]
pub enum DriverNameFormat {
    NameMiddleSurname,
    NameMiddleInitialSurname,
    NameSurname,
    InitialSurname,
    SurnameInitial,
    SurnameOnly,
}
```

---

## 3. Відповідність TS ↔ Rust (ключові поля)

| TS поле | Джерело iRSDK | Rust поле | Коментар |
|---|---|---|---|
| `Standings.carIdx` | `Driver.CarIdx` / `SessionResultsPosition.CarIdx` | `DriverRowVm.car_idx` | зберегти `i32` (native) |
| `driver.name` | `Driver.UserName` | `DriverStaticVm.name` | `Arc<str>` щоб уникати клонів |
| `driver.carNum` | `Driver.CarNumber` | `DriverStaticVm.car_num` | як `Arc<str>` |
| `driver.license` | `Driver.LicString` | `DriverStaticVm.license` | як `Arc<str>` |
| `driver.rating` | `Driver.IRating` | `DriverStaticVm.rating` | `i32` без downcast |
| `driver.flairId` | `Driver.FlairID` | `DriverStaticVm.flair_id` | `Option<i32>` |
| `driver.teamName` | `Driver.TeamName` | `DriverStaticVm.team_name` | `Option<Arc<str>>` |
| `carClass.*` | `Driver.CarClass*` | `CarClassVm.*` | `i32/f32/u32` |
| `delta/gap/interval` | розрахунок | `Option<f32>`/`Gap` | `f32` для мін. пам’яті |
| `lastTimeState` | computed | `Option<LastTimeState>` | enum |
| `flags` (dnf/repair/penalty/slowdown) | `carIdxSessionFlags` | `DriverFlags` | можливе bit‑packing |

---

## 4. Оптимізація ресурсів і типів

### 4.1. Типи, які **не варто** зменшувати
- `CarIdx`, `CarID`, `CarClassID`, `IRating` — у iRSDK це цілі числа. Якщо вони приходять як `i32`, зменшення до `u8/u16` потребує перевірок/кастів для кожного кадру.
- Через високу частоту оновлення UI краще **залишити `i32`**, якщо downcast не дає істотної економії.

### 4.2. Типи, які можна оптимізувати без значної вартості
- Прапори стану (`dnf`, `repair`, `penalty`, `slowdown`, `on_pit_road`, `on_track`, `radio_active`) — залишити як `bool` у `DriverFlags` для простоти, або перейти на `bitflags` якщо потрібна щільна упаковка.
- `LastTimeState` — `enum` (1 байт у Rust), без строкового формату.
- Колір класу: `u32` packed RGB (як у TS).
- Час/дельта: `f32` (iRSDK теж працює з float) → мінімум перетворень.

### 4.3. Статичні vs динамічні дані
Щоб мінімізувати алокації:
- `DriverStaticVm` створювати один раз на сесію (name, license, car, class).
- `DriverRowVm` оновлювати per‑tick лише по динамічним полям (delta, lap, flags).

Це суттєво зменшить кількість повторних алокацій рядків.

---

## 5. Рекомендації по конверсії з iRSDK

1. **Не копіювати великі структури.** Брати тільки потрібні поля (`Driver`, `SessionResultsPosition`).
2. **Кешувати статичні поля.** Наприклад `DriverStaticVm` будується лише коли змінюється склад драйверів.
3. **Уникати `String` у hot‑path.** Для імен/ліцензій — `Arc<str>` або `SmolStr` (якщо дозволена залежність, `SmolStr` вигідний для коротких рядків і зменшує алокації).
4. **Дельти та часи** тримати як `f32`, форматувати лише перед рендером (або один раз при побудові ViewModel).

---

## 6. Підсумок

- Основні TS‑моделі Standings зведені до компактних Rust ViewModel структур.
- Рекомендовано зберігати типи максимально близько до iRSDK (`i32`, `f32`), щоб не витрачати CPU на конверсію кожен кадр.
- Статичні та динамічні частини розділені для мінімізації алокацій і копіювань.
- ViewModel придатний для UI рендеру без зайвих залежностей чи важкої логіки.
