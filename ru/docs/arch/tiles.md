# Модель Тайла v0.2

> **Tile = минимальная программируемая сущность Decima-8**

---

## 📊 Структура Тайла

### Baked State (пропечённое)

| Параметр | Тип | Диапазон | Описание |
|----------|-----|----------|----------|
| **thr_lo16** | i16 | -32768..+32767 | Нижний порог фьюза |
| **thr_hi16** | i16 | -32768..+32767 | Верхний порог фьюза |
| **decay16** | u16 | 0..32767 | Затухание к нулю |
| **domain_id4** | u8 | 0..15 | Группа для сброса |
| **priority8** | u8 | 0..255 | Приоритет при коллизии |
| **pattern_id16** | u16 | 0..32767 | ID распознаваемого паттерна |
| **routing_flags16** | u16 | 10 бит | Направления активации |
| **W[8][8]** | SignedWeight5 | mag4+sign1 | Матрица весов 8×8 |
| **reset_on_fire_mask16** | u16 | 16 бит | Авто-сброс доменов при fire |

### Runtime (изменяемое)

| Параметр | Тип | Описание |
|----------|-----|----------|
| **thr_cur16** | i16 | Текущий аккумулятор (-32768..+32767) |
| **locked** | 0/1 | Фьюз защёлкнут (latched) |
| **drive_vec[8]** | u8[8] | Выходные значения (0..15) |

> **Примечание:** Decay применяется всегда (если decay16 > 0), даже на locked тайлах.

---

## 🔄 ACTIVE Closure (Граф Активации)

Тайл считается **ACTIVE** если он «в цепочке живой» и может читать BUS16, вычисляться, применять decay.

### Правила Активации

```
Seed: ACTIVE[t] = 1 если BUS_R == 1 (источник/корень цепочки)

Propagate: ACTIVE[t] = 1 если существует p ∈ Parents(t) такой что:
           ACTIVE[p] == 1 И locked_before[p] == 1
```

Это вычисляется как **монотонное замыкание до стабилизации** (least fixed point).

### Принудительный Схлоп Ветки

Если `ACTIVE[t] == 0`, тайл считается «мёртвым участком»:

```
thr_cur16 := 0
locked := 0
drive_vec := {0, 0, 0, 0, 0, 0, 0, 0}
```

Веса, decay, drive не применяются в этом tick.

---

## 📥 Вход Тайла (VSB_INGRESS)

Если `ACTIVE[t] == 1`, тайл читает **только VSB_INGRESS16** (все 8 lane):

```
for i in 0..7:
  in16[t][i] = clamp15(VSB_INGRESS16[i])
  IN_CLIP[t][i] = (VSB_INGRESS16[i] > 15)
```

> **Важно:** Шина BUS16 не суммируется с VSB. Единственная роль BUS16 в фазе READ — семантическая: тайлы с флагом BUS_R становятся источниками графа активации (ACTIVE seed).

### Эстафета (2 тика)

```
Тик N:   Предок фьюзится → драйвит шину в PHASE_WRITE
Тик N+1: Потомок активируется через BUS_R → читает VSB_INGRESS → вычисляется
```

---

## 🔒 FUSE-LOCK Механизм

### Decay-to-Zero + Fuse-by-Range

**Decay применяется всегда** (если decay16 > 0), даже на locked тайлах.

Если `locked_before == 0`:

```python
# 1. Считаем row-pipeline
delta_raw = Σ row16_signed[r]  # диапазон [-6720..+6720]

# 2. Обновляем аккумулятор
thr_tmp = thr_cur16 + delta_raw

# 3. Decay тянет к 0 и не перескакивает 0
if decay16 > 0:
  if thr_tmp > 0:
    thr_tmp = max(thr_tmp - decay16, 0)
  elif thr_tmp < 0:
    thr_tmp = min(thr_tmp + decay16, 0)

thr_cur16 = clamp_range(thr_tmp, -32768, 32767)

# 4. Фьюз по диапазону
range_active = (thr_lo16 < thr_hi16)
in_range = range_active AND (thr_lo16 <= thr_cur16) AND (thr_cur16 <= thr_hi16)

has_signal = (delta_raw != 0)
entered_by_decay = (decay16 > 0) AND (in_range == true) AND (in_range_before_decay == false)

locked_after = (BAKE_APPLIED == 1) AND in_range AND (has_signal OR entered_by_decay)
```

Если `locked_before == 1`:

```python
locked_after := 1
# Веса не применяются (passthrough)

# Decay применяется всегда (если decay16 > 0)
if decay16 > 0:
  if thr_cur16 > 0:
    thr_cur16 = max(thr_cur16 - decay16, 0)
  elif thr_cur16 < 0:
    thr_cur16 = min(thr_cur16 + decay16, 0)
```

### Locked Passthrough

Если `locked_after == 1`, тайл действует как «медный мост»:

- Матрица W **не применяется**
- `drive_vec[i] = in16[i]` для всех i=0..7 (passthrough)
- **Decay применяется** (если decay16 > 0, тянет thr_cur16 к 0)

### Latched State

Если `locked_before == 1`:

- `locked_after := 1`
- Веса не применяются (passthrough)
- **Decay применяется всегда** (если decay16 > 0)

```python
# Decay тянет к 0 даже на locked тайлах
if decay16 > 0:
  if thr_cur16 > 0:
    thr_cur16 = max(thr_cur16 - decay16, 0)
  elif thr_cur16 < 0:
    thr_cur16 = min(thr_cur16 + decay16, 0)
```

---

## 🧮 RowOut Pipeline (PHASE_READ)

Для каждой строки r=0..7:

### Signed Умножение

```
row_raw_signed[r] = Σ_{i=0..7} (in16[i] * Wmag[r][i] * sign)
# Диапазон: [-840..+840]
```

### Для Линий/Drive (без отрицательных)

```
row16_out[r] = clamp15((max(row_raw_signed[r], 0) + 7) / 8)
# Диапазон: 0..15
```

### Для Аккумулятора (signed)

```
row16_signed[r] = row_raw_signed[r]
# Диапазон: [-840..+840]
```

---

## 🚗 Drive Selection (WRITE)

В конце READ:

```
if locked_after == 1:
  drive_vec[i] = in16[i]  # passthrough
else:
  drive_vec[i] = row16_out[i]  # вычисленное
```

---

## 🎯 События Тайла

| Событие | Условие |
|---------|---------|
| **LOCK_TRANSITION(t)** | locked_before==0 && locked_after==1 |
| **FIRE(t)** | LOCK_TRANSITION(t) |

### Инвариант v0.2

```
locked == 1 ⇒ thr_lo16 <= thr_cur16 <= thr_hi16
```

---

## 📐 SignedWeight5

Вес: mag4 ∈ [0..15], sign1 ∈ {0,1} (1="+", 0="−")

```
mul_signed_raw(a, mag4, sign1) = (sign1 ? +1 : -1) * (a * mag4)
# a ∈ [0..15] → диапазон [-225..+225]
```

---

## 🧩 Пример Состояния

```json
{
  "baked": {
    "thr_lo16": 100,
    "thr_hi16": 200,
    "decay16": 5,
    "domain_id4": 0,
    "priority8": 128,
    "pattern_id16": 42,
    "routing_flags16": 0x0301,  // N + BUS_R
    "reset_on_fire_mask16": 0x0001
  },
  "runtime": {
    "thr_cur16": 150,
    "locked": 1
  }
}
```

---

**Bake the Future. Build the Substrate.** 🛠️⚡️
