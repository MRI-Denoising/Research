# MRI Denoising Research

## 1. 연구 개요

Multi-Echo GRE MRI 복소수 데이터를 대상으로 **6가지 MP-PCA 기반 디노이징 처리 방식**을 비교합니다. 인위적인 Gaussian noise를 추가한 후 원본 영상과 복원 영상의 차이를 RSNR로 평가하고, Echo별 수치 및 영상을 시각화했습니다.

## 2. 데이터 및 실험 조건

### 데이터 구조 (`01_Load_data.ipynb`)

| 항목 | 설명 |
|---|---|
| 파일 형식 | MATLAB `.mat` (`meas_gre_dir1.mat`) |
| MRI 배열 | `meas_gre`: **256 × 224 × 176 × 6** (X × Y × Z × Echo) |
| 자료형 | 복소수 (`complex128`) |
| Echo 수 | 6 |
| 뇌 영역 마스크 | `mask_brain`: 256 × 224 × 176 |
| 분석 대상 | Brain Mask 내부 voxel |

**복소 MRI 신호**

$$
S(x,y,z,e)=R(x,y,z,e)+jI(x,y,z,e)
$$

**Magnitude 영상**

$$
M(x,y,z,e)=|S(x,y,z,e)|=\sqrt{R(x,y,z,e)^2+I(x,y,z,e)^2}
$$

여기서 `R`은 실수부, `I`는 허수부, `e`는 Echo 번호입니다.

### Gaussian noise 생성 (`02_Add_noise.ipynb`)

원본 MRI의 실수부·허수부에 **서로 독립적인 정규분포 잡음**을 추가합니다.

$$
S_{\mathrm{noisy}}=(R+n_R)+j(I+n_I), \qquad n_R,n_I\sim\mathcal{N}(0,\sigma^2)
$$

$$
\sigma = r\,\overline M_{\mathrm{brain}}, \qquad r\in\{0.1,\ 0.3,\ 0.5\}
$$

- Noise 수준: **10%, 30%, 50%** (뇌 영역 내 Echo별 평균 Magnitude의 전체 평균에 대한 표준편차 비율)
- 난수 시드: `42`
- 잡음 추가 영역: Brain Mask 내부
- 생성 데이터: `noisy_real`, `noisy_imag` (`float32`), `noisy_complex` (`complex64`)

## 3. MP-PCA 기반 디노이징 방법

| 파일 | 방법 | 핵심 처리 방식 |
|---|---|---|
| `03_Magnitude.ipynb` | **Magnitude** | 복소수의 크기 영상 `abs(S)`에 MP-PCA 적용 |
| `04_Real_Imag.ipynb` | **Real/Imag Split** | 실수부와 허수부에 각각 MP-PCA를 적용한 후 재결합 |
| `05_Real_Imag_Combined.ipynb` | **Real/Imag Combined** | 6개 Echo의 Real·Imaginary를 **12채널**로 결합해 MP-PCA 적용 |
| `06_Direct_Complex.ipynb` | **Direct Complex** | 복소수 데이터를 직접 입력하여 처리 |
| `07_Complex_Wishart.ipynb` | **Complex Wishart** | 복소수 공분산 특성을 고려한 별도 MP-PCA 구현 활용 |
| `08_Hybrid_Multi_Echo.ipynb` | **Hybrid Multi-Echo** | Multi-Echo 신호의 특성을 함께 활용하는 Hybrid 방식 |

**Split 방식의 재결합**

$$
\widehat S=D(R)+jD(I)
$$

여기서 `D`는 각각의 데이터에 적용하는 디노이징 연산입니다.

**Combined 방식의 채널 구성**

$$
X=[R_1,R_2,\ldots,R_6,I_1,I_2,\ldots,I_6]
$$

원래의 입력이 `(X, Y, Z, 6)` 복소수 배열이라면, Combined 방식은 `(X, Y, Z, 12)` 실수 채널 배열로 바꾼 뒤 처리합니다.

> 참고: Complex Wishart 및 Hybrid Multi-Echo 방식은 별도 Python 구현 모듈에 의존합니다. 내부 연산식을 노트북만으로 완전히 확인할 수 없으므로 여기에는 입력·처리 관점의 개요만 기재했습니다.

## 4. 성능 평가 (`09_RSNR_Comparison_with_Visualization.ipynb`)

노이즈 추가 전 원본 **Magnitude 영상**을 기준(Reference)으로 하여 각 Echo의 **RSNR(Reference-based Signal-to-Noise Ratio)**을 계산했습니다. 모든 평균 값은 6개 Echo의 RSNR을 산술평균한 값입니다.

**RSNR 계산식**

$$
\mathrm{RSNR}=\frac{\sqrt{\frac{1}{N}\sum_{i=1}^{N}M_{\mathrm{ref},i}^{2}}}{\sqrt{\frac{1}{N}\sum_{i=1}^{N}(M_{\mathrm{test},i}-M_{\mathrm{ref},i})^{2}}+10^{-12}}
$$

- `M_ref`: 원본 Magnitude, `M_test`: Noisy 또는 Denoised Magnitude
- `N`: Brain Mask 내부 voxel 수
- **주의: 현재 코드의 RSNR은 선형 비율값이며 dB가 아닙니다.** 높은 값일수록 원본과의 복원 오차가 작습니다.

### 평균 RSNR 결과

| Method | Noise 10% | Noise 30% | Noise 50% |
|---|---:|---:|---:|
| Noisy | 10.86 | 3.63 | 2.20 |
| Magnitude | 18.47 | 8.00 | 4.52 |
| Real/Imag Split | 17.04 | 8.10 | 6.11 |
| **Real/Imag Combined** | **18.92** | **9.42** | **6.96** |
| Direct Complex | 17.99 | 8.35 | 6.27 |
| Complex Wishart | 18.00 | 8.45 | 6.38 |
| Hybrid Multi-Echo | 14.87 | 7.82 | 5.45 |

현재 실험 조건에서는 **Real/Imag Combined** 방식이 세 가지 noise 수준 모두에서 가장 높은 평균 RSNR을 보였습니다.

**RSNR 개선율 계산식**

$$
\mathrm{Improvement}(\%)=\frac{\mathrm{RSNR}_{\mathrm{denoised}}-\mathrm{RSNR}_{\mathrm{noisy}}}{\mathrm{RSNR}_{\mathrm{noisy}}}\times 100
$$

### 결과 시각화

09번 노트북에서 다음 결과를 확인할 수 있습니다.

- Echo별 RSNR 비교 및 noise 수준별 평균 RSNR 막대그래프
- 원본·Noisy·6가지 Denoised Magnitude 영상 비교
- 원본 영상 대비 Difference Map
- Noise 수준 × Echo별 RSNR Heatmap 및 방법별 순위

## 5. 노트북 구성 및 읽는 순서

1. `01_Load_data.ipynb` — 원본 데이터 및 Brain Mask 확인
2. `02_Add_noise.ipynb` — Gaussian noise 데이터 생성
3. `03_Magnitude.ipynb` ~ `08_Hybrid_Multi_Echo.ipynb` — 6가지 MP-PCA 방식 적용
4. `09_RSNR_Comparison_with_Visualization.ipynb` — 정량적 비교 및 시각화

**연구 결과를 먼저 확인하려면 `09_RSNR_Comparison_with_Visualization.ipynb`를 참고하면 됩니다.**
