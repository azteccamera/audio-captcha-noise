# 오디오 CAPTCHA 잡음 설계와 음성인식 공격 실험

숫자 6자리 오디오 CAPTCHA에 9종의 잡음을 여러 SNR로 섞은 뒤, Whisper large-v3 기반 공격자 5종의 공격 성공률과 객관적 명료도(STOI·ESTOI)를 비교한 실험의 코드와 결과입니다.

<!-- 논문 제목과 서지 정보가 정해지면 여기에 적어 주세요. -->

## 구성

| 경로 | 내용 |
|---|---|
| `00_digit_bank.ipynb` | edge-tts로 숫자 음성 60개(목소리 6개 × zero~nine) 합성 |
| `01_main_experiment.ipynb` | 본실험: CAPTCHA 생성 → 잡음 혼합 → 공격자 A1~A5 해독 → STOI·ESTOI → 논문의 표와 그림 |
| `02_main_separation.ipynb` | SepFormer 음성 분리 (A4·A5 공격자용) |
| `data/digits/normal/` | 실험에 실제로 사용한 숫자 음성 60개 |
| `runs/main/` | 본실험 결과 (조건당 200개) |

## 공격자와 잡음 이름

| 코드 | 논문 | 설정 |
|---|---|---|
| `base` | A1 기본 | 온도 0, 재시도 없음, 비음성 판정 기본값 |
| `hintX` | A2 디코딩 조정 | 온도 폴백(0~1.0), 비음성 판정 끔 |
| `hintO` | A3 디코딩 조정 + 프롬프트 | A2 + 'six random digits'를 알려 주는 초기 프롬프트 |
| `sep_base` | A4 분리 | 음성 분리 후 두 출력에 A1 적용 |
| `sep_hintX` | A5 분리 + 디코딩 조정 | 음성 분리 후 두 출력에 A2 적용 |

잡음: `white` 백색 잡음, `ssn` 어음스펙트럼 잡음, `band_low`·`band_mid`·`band_high` 대역 제한 잡음(저역·중역·고역), `talk1`·`talk2`·`talk4` 경쟁 화자 1·2·4명, `babble8` 다화자 잡음(8명)

## 논문의 표·그림과 결과 파일

모두 `runs/main/results/`에 있으며 `01_main_experiment.ipynb`의 6번 셀들이 만듭니다.

| 논문 | 파일 | 셀 |
|---|---|---|
| 표 3 | `table3_best_attacker_rate.csv` (공격자별 값: `table_rate_by_attacker.csv`) | 6-1 |
| 그림 1 | `fig_final.png` | 6-2 |
| 표 4 | `table4_separation_effect.csv` | 6-3 |
| 표 5 | `table_rq3_estoi_threshold.csv` | 6-4 |
| 표 6 | `table6_failure_types_pct.csv` (세부 유형별 개수: `table_failure_types.csv`) | 6-5 |
| 3.4절 클리핑 점검 | 노트북 출력 | 6-6 |

그 밖의 파일

- `summary_final.csv`: 조건별 가장 강한 공격자, 공격 성공률과 95% 신뢰구간(Wilson), 평균 STOI·ESTOI
- `asr_<공격자>.csv`: 클립별 출력 (`raw` 음성인식 모델의 출력 문자열, `hyp` 출력에서 추출한 숫자열, `first6` 앞 6자리 일치 = 공격 성공, `exact` 완전 일치 채점 기준의 정답 여부)
- `intelligibility.csv`: 클립별 STOI·ESTOI
- `runs/main/clean.csv`, `mixed.csv`, `separated.csv`: CAPTCHA 정답과 목소리, 잡음 혼합 음성 목록(최대 진폭 `peak` 포함), 음성 분리 결과 목록

## 결과만 다시 만들기 (GPU 불필요)

```bash
pip install numpy pandas matplotlib soundfile tqdm
```

`01_main_experiment.ipynb`에서 **0번 셀과 6번 셀들만** 실행하면 위의 표와 그림이 다시 만들어집니다.

## 처음부터 다시 실행하기

### 환경

Windows 11, NVIDIA GPU(8GB), Python 3.11에서 실행했습니다. 전체 실행에는 약 14시간과 디스크 여유 공간 약 10GB가 필요합니다.

### 설치

```bash
pip install -r requirements.txt
# PyTorch와 torchaudio는 CUDA 버전에 맞게 https://pytorch.org 에서 설치 (실험: 2.10.0 nightly, CUDA 12.8)
pip install --no-deps speechbrain==1.1.2   # 설치한 PyTorch를 덮어쓰지 않도록 --no-deps
```

### 데이터 준비

1. MUSAN([OpenSLR 17](https://www.openslr.org/17/))을 내려받아 `speech` 폴더를 `data/noise/musan/speech/`에 둡니다(wav 426개).
2. Whisper large-v3와 SepFormer(`speechbrain/sepformer-whamr16k`)는 처음 실행할 때 자동으로 내려받습니다.

### 실행 순서

자세한 내용은 `01_main_experiment.ipynb` 맨 위의 안내를 참고하세요.

1. (선택) `00_digit_bank.ipynb`: 숫자 음성이 이미 들어 있으므로 다시 실행하지 않아도 됩니다.
2. `01_main_experiment.ipynb`를 Run All (3-3 해독 약 7시간, 5 명료도 약 1시간)
3. 커널을 끄고 `02_main_separation.ipynb` 실행 (약 30분)
4. 커널을 끄고 `01_main_experiment.ipynb`에서 0 → 3-1 → 3-2 → 4 → 6 순서로 실행 (4번 셀 약 5시간)

0번 셀에서 `DRY_RUN = True`로 바꾸면 조건당 5개로 전체 과정을 빠르게 시험해 볼 수 있습니다.

## 재현성

- 무작위 과정은 시드 42로 고정했습니다. 다만 GPU와 라이브러리 버전에 따라 출력이 조금 달라질 수 있습니다.
- edge-tts(Microsoft Edge 온라인 TTS)는 같은 음성이 다시 만들어진다는 보장이 없어서, 실험에 쓴 숫자 음성 60개를 저장소에 포함했습니다.
- 실험 중 생성되는 WAV(깨끗한 음성, 잡음 혼합 음성, 분리된 음성)와 외부 데이터·모델은 용량 때문에 포함하지 않았습니다(`.gitignore` 참고). 외부 데이터와 모델은 각 출처의 라이선스를 따릅니다.
- 실제 서비스 중인 CAPTCHA는 대상으로 삼지 않았습니다.
