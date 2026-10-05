可以。這版整理為：**保留內側校準結果，讓使用者用一個 `Vmodel`，連動調整外側斜率與偏移；韌體重新計算交點，保持兩段連續。**

以下是確認稿，先不回寫 HTML。這是自研的調參設計，尚未確認 ACS 內部採用相同方式。

## 1. 這個方案要解決什麼？

假設外側有兩個工作點：

- A 點已經對齊。
- B 點仍有偏差，表示 A／B 的電流跨距需要調整。

直接改變整體倍率，通常會讓 A 點一起移動。因此，本版的調整方式是：

**以已對齊的 A 點為支點，調整外側直線的斜率；偏移量同步改變，讓直線仍然通過 A 點。**

使用者只調整 `Vmodel`。內部的 `Kv`、`Vbias` 由韌體連動計算。

這個 A 點是「選定的外側校準點」，不一定是先前提到的零電流位置 A。

## 2. 參數如何定義？

| 參數 | 意義 | 設定方式 |
|---|---|---|
| `V_ref` | 將實寫 CMP 換算成基準電壓的固定值 | 校準時決定，例如 `96 V` |
| `Vmodel` | 使用者的外側跨距調整旋鈕 | 初始值設為 `V_ref` |
| `Kv0` | 基準平台的外側斜率 | 校準後保存 |
| `Vbias0` | 基準平台的外側偏移，單位 V | 校準後保存 |
| `K_in0` | 基準平台的內側斜率，通過原點 | 校準後保存 |
| `u_A0` | 已對齊外側 A 點的基準電壓，單位 V | 校準後保存 |
| `Ub` | 內、外兩段的交點幅度，單位 V | 由韌體計算 |

前面討論的 `V0` 可以直接等於 `V_ref`，所以本版不用另外保存 `V0`。

基準電壓仍然使用固定的 `V_ref`：

```c
float m = ((float)cmp_u_applied - (float)cmp_w_applied)
        * (1.0f / 5000.0f);

float u = V_ref * m;
```

兩路 CMP 是**對應作用區間、實際寫入的整數絕對比較值**。

`Vmodel` 在本版是校準旋鈕；調整它不代表實際母線電壓改變。

## 3. `Vmodel` 如何連動 `Kv` 與 `Vbias`？

先求出調整比例：

$$
\rho=\frac{V_{\mathrm{model}}}{V_{\mathrm{ref}}}
$$

外側斜率跟著比例改變：

$$
K_v=\rho K_{v0}
$$

偏移則同步調整：

$$
V_{\mathrm{bias}}
=
V_{\mathrm{bias}0}
+
\left(K_{v0}-K_v\right)u_{A0}
$$

為什麼偏移要這樣計算？

原本 A 點的校正電壓是：

$$
u_{\mathrm{cal},A}
=
K_{v0}u_{A0}+V_{\mathrm{bias}0}
$$

調整斜率後，希望同一個 A 點仍然得到相同電壓，因此要求：

$$
K_vu_{A0}+V_{\mathrm{bias}}
=
K_{v0}u_{A0}+V_{\mathrm{bias}0}
$$

移項後，就是上面的偏移公式。

所以外側直線也能寫成：

$$
u_{\mathrm{cal}}(u)
=
u_{\mathrm{cal},A}
+
\rho K_{v0}(u-u_{A0})
$$

這條公式直接表達：

- `u = u_A0`：A 點不變。
- 距離 A 點越遠：受到斜率調整的影響越大。
- `Vmodel` 增加：外側跨距增加。
- `Vmodel` 減少：外側跨距縮小。

程式對應：

```c
rho   = Vmodel / V_ref;
Kv    = rho * Kv0;
Vbias = Vbias0 + (Kv0 - Kv) * u_A0;
```

**本版不要再於輸出乘一次 `Vmodel / V_ref`。** 調整比例已經放進 `Kv` 與 `Vbias`，再乘一次會重複縮放。

## 4. 外側調整後，內側怎麼辦？

本版保持內側斜率 `K_in0` 不變：

$$
u_{\mathrm{cal}}=K_{\mathrm{in}0}u
$$

外側使用新斜率與偏移：

$$
u_{\mathrm{cal}}=K_vu+V_{\mathrm{bias}}
$$

兩條直線重新求交點，讓接合處連續。

負向交點在 `u = −Ub_neg`：

$$
U_{b,\mathrm{neg}}
=
\frac{V_{\mathrm{bias,neg}}}
{K_{v,\mathrm{neg}}-K_{\mathrm{in,neg}0}}
$$

正向交點在 `u = +Ub_pos`：

$$
U_{b,\mathrm{pos}}
=
\frac{V_{\mathrm{bias,pos}}}
{K_{\mathrm{in,pos}0}-K_{v,\mathrm{pos}}}
$$

選定正、負側後，完整電壓映射為：

$$
\hat u=
\begin{cases}
K_{\mathrm{in}0}u,
& |u|\le U_b\\[4pt]
K_vu+V_{\mathrm{bias}},
& |u|>U_b
\end{cases}
$$

這樣具有三個特性：

1. `u = 0` 時，校正電壓也是零。
2. 內側斜率保持原校準值。
3. 內、外兩段在交點接合，沒有電壓跳躍。

要注意：**交點移動後，轉折附近的映射也會改變。**「內側不變」指的是仍然落在新內側範圍中的工作點。

`Ub` 是兩條直線的交點；它是否仍符合實測的 deadtime 附近轉折，需要另外確認。

## 5. 帶入完整數值範例

以下沿用前幾輪的示例，並假設內側重新擬合後得到 `0.53`：

```text
V_ref   = 96 V
R       = 10 Ω

K_in0   = 0.53
Kv0     = 0.90
Vbias0  = 1.389 V

u_A0    = −5 V
u_B     = −6 V
```

基準 A 點：

$$
u_{\mathrm{cal},A}
=
0.90(-5)+1.389
=
-3.111\ \mathrm{V}
$$

穩態估測電流為：

$$
\hat i_A
=
\frac{-3.111}{10}
=
-0.3111\ \mathrm{A}
$$

現在把 `Vmodel` 增加 5%，設為 `100.8 V`：

$$
\begin{aligned}
\rho&=1.05\\
K_v&=1.05\times0.90=0.945\\
V_{\mathrm{bias}}
&=1.389+(0.90-0.945)(-5)\\
&=1.614\ \mathrm{V}
\end{aligned}
$$

結果如下：

| 項目 | `91.2 V`，−5% | `96 V`，基準 | `100.8 V`，＋5% |
|---|---:|---:|---:|
| 外側 `Kv` | 0.855 | 0.900 | 0.945 |
| 外側 `Vbias` | 1.164 V | 1.389 V | 1.614 V |
| 內側 `K_in` | 0.530 | 0.530 | 0.530 |
| 交點幅度 `Ub` | 3.582 V | 3.754 V | 3.889 V |
| 內側 `u = −1 V` 的估測電流 | −0.0530 A | −0.0530 A | −0.0530 A |
| A 點 `u = −5 V` 的估測電流 | −0.3111 A | −0.3111 A | −0.3111 A |
| B 點 `u = −6 V` 的估測電流 | −0.3966 A | −0.4011 A | −0.4056 A |
| A／B 電流跨距 | 85.5 mA | 90.0 mA | 94.5 mA |

因此，如果 A 已對齊、B 還不夠負，增加 `Vmodel` 可以把 B 往更負的方向調整，同時保留 A。

## 6. 實作範例：CPU 算係數，CLA 執行映射

下面分成兩部分。CPU 在初始化或調參時處理除法與檢查；CLA 每拍只需要判斷、乘法與加法。

### CPU：更新校準參數

正、負側各保存一組基準值：

```c
#include <math.h>

typedef struct {
    float Kin0;
    float Kv0;
    float bias0_V;
    float u_anchor_V;
} VCalBase;

typedef struct {
    float Kin;
    float Kv;
    float bias_V;
    float Ub_V;
} VCalRun;
```

以下函式可共用於正、負側：

```c
// direction：正側 +1.0f，負側 -1.0f。
// 成功回傳 1；失敗時不修改 out。
// 此函式在 CPU 執行。
static int VCal_Prepare(
    VCalRun *out,
    const VCalBase *base,
    float rho,
    float direction)
{
    VCalRun next;

    if (!isfinite(rho) || rho <= 0.0f)
        return 0;

    next.Kin    = base->Kin0;
    next.Kv     = rho * base->Kv0;
    next.bias_V = base->bias0_V
                + (base->Kv0 - next.Kv) * base->u_anchor_V;

    float denominator = next.Kin - next.Kv;

    if (!isfinite(next.Kin) || next.Kin <= 0.0f ||
        !isfinite(next.Kv)  || next.Kv  <= 0.0f ||
        !isfinite(next.bias_V) ||
        denominator == 0.0f)
        return 0;

    next.Ub_V = direction * next.bias_V / denominator;

    // 交點須有限且為正，保存的支點須仍位於外側。
    if (!isfinite(next.Ub_V) || next.Ub_V <= 0.0f ||
        !isfinite(base->u_anchor_V) ||
        direction * base->u_anchor_V <= next.Ub_V)
        return 0;

    *out = next;
    return 1;
}
```

帶入範例：

```c
const float V_ref = 96.0f;

// 負側示例。
const VCalBase neg_base = {
    0.53f, 0.90f, 1.389f, -5.0f
};

// 正側暫用鏡射示例；實際應由正向資料校準。
const VCalBase pos_base = {
    0.53f, 0.90f, -1.389f, 5.0f
};

float rho = Vmodel / V_ref;

VCalRun neg_next;
VCalRun pos_next;

int valid =
    VCal_Prepare(&neg_next, &neg_base, rho, -1.0f) &&
    VCal_Prepare(&pos_next, &pos_base, rho, +1.0f);

if (valid) {
    // 在既有 CPU／CLA 安全同步點，
    // 一次發布完整的正、負側參數。
}
```

每次都從 `neg_base`、`pos_base` 重算，避免反覆調參造成累積縮放。

### CLA：每拍重建電壓

以下使用已完整發布的 `neg_run`、`pos_run`：

```c
float m = ((float)cmp_u_applied - (float)cmp_w_applied)
        * (1.0f / 5000.0f);

float u = V_ref * m;

const VCalRun *cal;
float abs_u;

if (u >= 0.0f) {
    cal   = &pos_run;
    abs_u = u;
} else {
    cal   = &neg_run;
    abs_u = -u;
}

float u_hat;

if (abs_u <= cal->Ub_V) {
    u_hat = cal->Kin * u;
} else {
    u_hat = cal->Kv * u + cal->bias_V;
}
```

這裡依**基準電壓的正負號**選擇映射。這是目前的靜態電壓校準方式，不另外引入量測電流。

### 接回原本的後向歐拉 RL 模型

CPU 先計算：

```c
// R：Ω；L_mH：mH；Ts_ms：ms。
float denominator = L_mH + R * Ts_ms;

a = L_mH / denominator;
b = Ts_ms / denominator;
```

CLA 使用對應已作用區間的 `u_hat` 更新：

```c
iq1 = a * iq1 + b * u_hat;
```

對應公式：

$$
\hat i[k+1]
=
a\hat i[k]+b\hat u[k]
$$

`iq1` 保留為下一拍狀態，不因零命令、方向切換或調參而清零。這個校準只改變模型使用的電壓，不修改實際 PWM，也不增加電流誤差修正支路。

## 7. 本版可以處理的範圍

這個單旋鈕版本適合：

**內側已準確、外側 A 點已對齊，但外側跨距仍需微調。**

它仍有兩個明確界線：

- 新平台如果連 A 點都偏了，固定 A 的旋鈕無法同時修正新的整體偏移。
- 新平台如果內側斜率也不同，本版保持 `K_in0` 不變，無法修正內側誤差。

所以，「使用者只調 `Vmodel`」成立的前提，是平台差異主要符合上述外側跨距變化。更換馬達時，仍需設定合理的 `R`、`L`。

下一步最小驗證：調整 `Vmodel` 後，確認**一個內側點、外側 A／B，以及转折附近**是否符合預期。新映射先旁路比較，再評估接入回授。

本輪數值檢查已確認支點固定、跨距比例、交點連續及 RL 穩態結果；程式骨架尚未經你的 CLA 編譯與實機驗證。