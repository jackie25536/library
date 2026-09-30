可以。這個做法可以把它理解成：

> **以原始馬達電氣參數與原始 Vbus 為基準，RL 模型本體固定；使用者後續調整 SLMR / SLVBUS 時，不直接把它們當成新的 R、Vbus，而是把「相對基準值的變化量」轉換成模型輸入電壓的 slope 與 offset 修正。**

它的目的，就是解決你現在遇到的：

\[
iq1=-0.22\sim-0.35
\]

但實際：

\[
iq=-0.16\sim-0.45
\]

這種**中心差不多，但 span 明顯不對**，而且單純調 R / Vbus 永遠只能整體縮放的問題。

---

# 1. 先定義基準狀態

PLUS 啟動前，保存最初的參數：

\[
\boxed{
R_0,\quad L_0,\quad V_0
}
\]

例如：

\[
R_0=4\Omega
\]

\[
L_0=2mH
\]

\[
V_0=48V
\]

這三個參數代表：

> 「還沒有做 PLUS 額外修正時的 nominal motor model。」

PWM 實際 modulation：

\[
\boxed{
m=
\frac{CMP_U-CMP_W}{TBPRD}
}
\]

基準模型電壓：

\[
\boxed{
u_0=V_0m
}
\]

所以原始 RL：

\[
\boxed{
L_0\frac{d\hat i}{dt}
=
u_0-R_0\hat i
}
\]

這就是你目前已經成功改善 jitter 的版本。

---

# 2. 為什麼不直接調 R / Vbus？

如果直接：

\[
u=SLVBUS\cdot m
\]

以及：

\[
L\frac{d\hat i}{dt}=u-SLMR\hat i
\]

穩態：

\[
\hat i\approx
\frac{SLVBUS}{SLMR}m
\]

所以不管調 R 還是 Vbus，本質都很像：

\[
\boxed{
\hat i\rightarrow K\hat i
}
\]

也就是繞著 0 A 做縮放。

這正是你的問題。

例如：

\[
-0.22\rightarrow-0.16
\]

需要：

\[
K=0.727
\]

那：

\[
-0.35\rightarrow-0.255
\]

所以另一端一定跑掉。

這不是你 R 沒調對，而是：

\[
\boxed{
一個 gain 根本不足以描述兩個工作點
}
\]

---

# 3. PLUS 修正改成 affine transformation

我們不再直接修改 RL 參數。

而是在：

\[
u_0=V_0m
\]

之後，加一層：

\[
\boxed{
u_{eff}=Au_0+B
}
\]

這就是整個方法的核心。

其中：

### A：斜率修正

控制：

\[
\boxed{\text{span / peak-to-peak}}
\]

### B：偏移修正

控制：

\[
\boxed{\text{center / offset}}
\]

然後 RL 本體仍然使用：

\[
\boxed{
L_0\frac{d\hat i}{dt}
=
u_{eff}-R_0\hat i
}
\]

因此：

\[
\boxed{
CMP
\rightarrow m
\rightarrow V_0m
\rightarrow Au_0+B
\rightarrow RL
\rightarrow iq1
}
\]

---

# 4. SLMR / SLVBUS 不直接進 RL，而是變成 A、B

這正是你前面提出的想法。

定義：

\[
\boxed{
d_V=
\frac{SLVBUS-V_0}{V_0}
}
\]

\[
\boxed{
d_R=
\frac{SLMR-R_0}{R_0}
}
\]

如果使用者完全沒有調：

\[
SLVBUS=V_0
\]

所以：

\[
d_V=0
\]

同理：

\[
SLMR=R_0
\Rightarrow d_R=0
\]

這時：

\[
A=1
\]

\[
B=0
\]

所以：

\[
u_{eff}=u_0
\]

完全回到你目前的原始 RL 模型。

---

# 5. SLVBUS 負責 A

定義：

\[
\boxed{
A=1+K_Vd_V
}
\]

所以：

\[
A
=
1+
K_V
\frac{SLVBUS-V_0}{V_0}
\]

若：

\[
SLVBUS=V_0
\]

則：

\[
A=1
\]

若 SLVBUS 增加：

\[
A>1
\]

估測電流 span 變大。

若 SLVBUS 減少：

\[
A<1
\]

span 變小。

所以：

\[
\boxed{
SLVBUS主要變成「span adjustment」
}
\]

---

# 6. SLMR 負責 B

定義：

\[
\boxed{
B=
K_Rd_RV_N
}
\]

其中：

\[
V_N
\]

只是 normalization voltage。

第一版可以簡單使用：

\[
V_N=V_0
\]

因此：

\[
\boxed{
B=
K_R
\frac{SLMR-R_0}{R_0}
V_0
}
\]

若：

\[
SLMR=R_0
\]

則：

\[
B=0
\]

修改 SLMR 時：

\[
B\neq0
\]

所以整條 iq1 波形會向上或向下移。

也就是：

\[
\boxed{
SLMR主要變成「center adjustment」
}
\]

---

# 7. 完整公式

最後整個模型：

\[
m=
\frac{CMP_U-CMP_W}{TBPRD}
\]

\[
u_0=V_0m
\]

\[
d_V=
\frac{SLVBUS-V_0}{V_0}
\]

\[
d_R=
\frac{SLMR-R_0}{R_0}
\]

\[
A=1+K_Vd_V
\]

\[
B=K_Rd_RV_0
\]

所以：

\[
\boxed{
u_{eff}
=
(1+K_Vd_V)V_0m
+
K_Rd_RV_0
}
\]

最後：

\[
\boxed{
L_0\frac{diq1}{dt}
=
u_{eff}
-
R_0iq1
}
\]

這就是第一版完整架構。

---

# 8. 用你的兩工作點理解

你目前：

\[
iq1=-0.22\sim-0.35
\]

中心：

\[
C_{iq1}
=
\frac{-0.22-0.35}{2}
=-0.285
\]

span：

\[
S_{iq1}=0.13
\]

實際：

\[
iq=-0.16\sim-0.45
\]

中心：

\[
C_{iq}
=-0.305
\]

span：

\[
S_{iq}=0.29
\]

所以問題其實是：

\[
\boxed{
center只差0.02A
}
\]

但：

\[
\boxed{
span差很多
}
\]

因此第一步應該主要調：

\[
A
\]

理想上：

\[
A
\approx
\frac{0.29}{0.13}
\approx2.23
\]

先把：

\[
0.13
\]

拉成：

\[
0.29
\]

然後再利用 B，把中心從修正後的位置移到：

\[
-0.305A
\]

這就是：

\[
\boxed{
先 span，後 center
}
\]

---

# 9. 實際調整方式

你之後在公司可以非常直覺。

先保持：

\[
SLMR=R_0
\]

所以：

\[
B=0
\]

只調 SLVBUS。

觀察：

\[
\boxed{
iq1的P2P是否接近iq的P2P
}
\]

例如：

原本：

\[
iq1=-0.22\sim-0.35
\]

慢慢變成：

\[
-0.15\sim-0.44
\]

這時 span 已經：

\[
0.29
\]

幾乎正確。

但中心：

\[
\frac{-0.15-0.44}{2}
=-0.295
\]

而目標：

\[
-0.305
\]

只差：

\[
-0.01A
\]

接著才調 SLMR，利用 B 把整條：

\[
-0.15\sim-0.44
\]

往下移：

\[
-0.16\sim-0.45
\]

就完成。

---

# 10. 這個方式最大的優點

你會得到類似兩個互相接近 orthogonal 的旋鈕：

\[
\boxed{
SLVBUS\rightarrow span
}
\]

\[
\boxed{
SLMR\rightarrow center
}
\]

這就比目前：

\[
SLVBUS\rightarrow gain
\]

\[
SLMR\rightarrow也是gain
\]

好調很多。

你現在原始 RL 最大的問題就是：

> R 與 Vbus 都在做很接近的事情。

因為穩態：

\[
i\propto\frac{Vbus}{R}
\]

所以它們其實高度耦合。

而這個 PLUS-style implementation 是刻意把它們變成：

\[
\boxed{
兩個不同功能的 tuning knob
}
\]

這就是為什麼我認為它有機會產生你描述的 ACS 那種：

> 「試個幾次就知道該往哪個方向調。」

---

# 11. KV 與 KR 是什麼？

這兩個不是馬達參數。

它們是：

\[
\boxed{
PLUS內部tuning sensitivity
}
\]

例如：

\[
K_V=1
\]

表示 SLVBUS 改 10%，A 就改 10%。

若你覺得太敏感：

\[
K_V=0.5
\]

SLVBUS 改 10%，A 只改 5%。

同理：

\[
K_R
\]

決定 SLMR 改變多少會產生多少 B。

所以：

\[
K_V,\ K_R
\]

應該固定在 firmware。

使用者完全不用看到。

---

# 12. 我會先怎麼設定 KV？

第一版可以：

\[
\boxed{K_V=1}
\]

先確認 concept。

這時：

\[
A
=
1+
\frac{SLVBUS-V_0}{V_0}
\]

所以：

\[
A=\frac{SLVBUS}{V_0}
\]

但要注意：

這雖然數學上跟直接 Vbus scaling 一樣，但現在 RL 使用的是固定：

\[
R_0,L_0,V_0
\]

而且我們額外加入了 B。

真正新增的自由度其實來自：

\[
\boxed{B}
\]

所以第一版真正想驗證的是：

> 加上 B 後，SLVBUS 的 gain 能不能終於把 span 調好，而 SLMR 再把 center 拉回來。

如果有效，再進一步把：

\[
K_V\neq1
\]

調成更適合使用者的手感。

---

# 13. KR 怎麼決定？

這不能一開始猜太大。

例如先設定：

\[
K_R=0.01
\]

或更小。

假設：

\[
R_0=4\Omega
\]

SLMR 改：

\[
4\rightarrow4.4
\]

所以：

\[
d_R=0.1
\]

若：

\[
V_0=48V
\]

\[
K_R=0.01
\]

則：

\[
B
=
0.01\times0.1\times48
\]

\[
B=0.048V
\]

若 R 大約 4 Ω，穩態相當於：

\[
\Delta i\approx\frac{0.048}{4}
=12mA
\]

這樣就非常好理解。

如果移動太小：

增加 KR。

如果太敏感：

降低 KR。

---

# 14. 正負方向怎麼做？

你現在先測的都是負電流：

\[
-0.16\sim-0.45
\]

所以第一版：

\[
\boxed{
B=K_Rd_RV_0
}
\]

即可。

不要先加方向。

先證明：

\[
A+B
\]

能解決兩工作點。

成功後才處理正電流。

---

## 第二版才做方向性

如果發現正方向需要相反 B：

\[
\boxed{
B=
K_Rd_RV_0S_i
}
\]

其中：

\[
S_i=
\operatorname{sgn}(iq1)
\]

所以：

\[
u_{eff}
=
Au_0
+
BS_i
\]

建議用 hysteresis：

\[
iq1>I_{th}
\Rightarrow S_i=+1
\]

\[
iq1<-I_{th}
\Rightarrow S_i=-1
\]

中間保持上一方向。

避免零附近 chatter。

---

# 15. L 怎麼處理？

第一階段：

\[
\boxed{L=L_0固定}
\]

不要讓 SLML 介入。

因為你目前處理的是：

\[
\boxed{\text{穩態與P2P span/center}}
\]

L 主要影響：

\[
\boxed{
\tau=\frac{L}{R}
}
\]

也就是：

- transient
- phase
- current rise/fall
- dynamic response

等你 A、B 可以把穩態對齊，再讓 SLML 去調 dynamic。

這樣 tuning 邏輯會非常清楚：

\[
\boxed{
SLVBUS\rightarrow span
}
\]

\[
\boxed{
SLMR\rightarrow offset
}
\]

\[
\boxed{
SLML\rightarrow dynamic
}
\]

這個使用體驗確實會非常漂亮。

---

# 16. 第一版程式概念

可以非常簡單：

```c
// Saved when PLUS is initialized
R0    = Motor_R_Init;
L0    = Motor_L_Init;
V0    = Vbus_Init;

// actual PWM
m = ((float)CMP_U - (float)CMP_W) / TBPRD;

// baseline voltage
u0 = V0 * m;

// relative user tuning
dV = (SLVBUS - V0) / V0;
dR = (SLMR   - R0) / R0;

// PLUS internal mapping
A = 1.0f + KV * dV;
B = KR * dR * V0;

// corrected voltage
u_eff = A * u0 + B;

// RL estimator
diq1 = (u_eff - R0 * iq1) / L0;

iq1 += Ts * diq1;
```

第一版就這樣。

不要再多加任何東西。

---

# 17. 你怎麼判斷這個方案成功？

我會設定三個主要判準：

1. 調 SLVBUS 時，`iq1` 的 P2P 明顯變化，而中心變化相對較小。
2. 調 SLMR 時，`iq1` 主要上下平移，而 P2P 不要大幅改變。
3. 最後可以同時讓：
   \[
   iq1_A\approx iq_A
   \]
   和：
   \[
   iq1_B\approx iq_B
   \]

而且：

\[
\boxed{
position\ jitter不能比你現在的CMP+fixed\ Vbus+RL基準差
}
\]

這一點非常重要。

因為我們現在已經有一個「jitter 確實改善」的 baseline。

任何新的 correction：

> 即使 iq1 更貼 iq，但 jitter 反而變差，都不能算成功。

---

# 18. 如果成功，這個方法的設計哲學會很乾淨

最終使用者看到：

\[
SLMR,\ SLML,\ SLVBUS
\]

但 firmware 內部其實不是單純：

\[
R=SLMR
\]

\[
L=SLML
\]

\[
V=SLVBUS
\]

而是：

\[
\boxed{
SLVBUS
\rightarrow voltage\ slope
}
\]

\[
\boxed{
SLMR
\rightarrow voltage\ bias
}
\]

\[
\boxed{
SLML
\rightarrow model\ dynamics
}
\]

馬達真正 nominal parameter：

\[
R_0,L_0,V_0
\]

則保持作為基準。

這確實就是你最開始提出的：

> 「是不是 ACS 會以初始電氣參數為基準，使用者調 SLMR / SLVBUS 時，內部再依參數差異做一些手腳？」

的一個**可以直接實作、可以直接驗證，而且不需要 LUT / polynomial 的版本。**

我會先用這個最小版本做驗證：

\[
\boxed{
u_{eff}
=
\left[
1+
K_V
\frac{SLVBUS-V_0}{V_0}
\right]
V_0m
+
K_R
\frac{SLMR-R_0}{R_0}
V_0
}
\]

先只測負電流 P2P。如果它真的讓你的 `span` 與 `center` 可以分別調整，那這條路就非常值得往下做。