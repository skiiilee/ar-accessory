# 웹 AR 안경 프로젝트 인수인계

## 개요
브라우저 카메라로 얼굴을 인식해 3D 안경을 실시간으로 씌우는 웹 AR 프로토타입이다. 손가락으로 브릿지를 잡아 안경을 위아래로 옮길 수 있다.

## 파일 구성 (같은 폴더에 함께 둘 것)
| 파일 | 설명 |
|---|---|
| `index.html` | 전체 앱 (단일 파일) |
| `glasses_v2.obj` | 앱에서 쓰는 안경 모델 |
| `glasses_v2.bak.obj` | 모델 백업 (앱에서 사용하지 않음) |

## 실행 방법
카메라는 HTTPS 또는 localhost에서만 동작한다.

- **PC:** 폴더에서 `python -m http.server 8000` 실행 후 `http://localhost:8000` 접속
- **모바일:** 로컬 서버를 켠 상태에서 `cloudflared tunnel --url http://localhost:8000` 실행 후, 출력된 `https://xxxx.trycloudflare.com` 주소로 접속

로컬 서버와 터널을 모두 켜 두어야 한다. 서버가 꺼지면 502 Bad Gateway가 표시된다.

## 현재 기능

### 안경 착용
| 항목 | 동작 |
|---|---|
| 위치 | 브릿지 중심을 콧등 랜드마크(6)에 맞춘다. 고개를 돌려도 브릿지가 얼굴 중앙에 있다 |
| 크기 | 눈 바깥 모서리(33, 263)의 3D 거리 × `frameRatio` ÷ 안경 앞면 폭(102.8mm). 고개 방향이 바뀌어도 크기를 유지한다 |
| 회전 | 얼굴 변환 행렬(`facialTransformationMatrixes`)을 기본으로 사용한다. `poseSource: "landmark"`로 랜드마크 근사 방식을 쓸 수 있다 |
| 흔들림 보정 | 위치, 회전, 크기에 `smoothing`을 적용한다 |

### 안경 다리
- 모델을 불러올 때 테를 앞면과 좌우 다리로 분리한다 (z -15mm 기준).
- 매 프레임 다리의 가장 높은 지점(z 약 -130mm)이 귀 위 기준점(랜드마크 127, 356)을 지나도록 힌지에서 위아래로 회전한다.
- 다리가 얼굴 윤곽 바깥을 지나도록 힌지에서 바깥쪽으로 벌린다.

### 얼굴 가림
- 얼굴 468점 메시를 깊이만 기록하는 보이지 않는 메시로 그린다.
- 렌즈 평면보다 앞에 있는 얼굴 점은 평면 뒤로 밀어서, 안경 앞면은 항상 보인다.
- 메시의 좌우 폭을 다리 안쪽까지로 제한한다. 다리는 힌지부터 귀 쪽까지 이어져 보이고, 고개를 돌리면 반대쪽 다리만 얼굴에 가려진다.

### 손으로 안경 옮기기
- 검지 또는 엄지 끝이 브릿지 근처에 오면 잡은 상태로 판단하고, 손가락이 움직인 만큼 안경을 옮긴다. 범위는 0~10mm이다.
- 손을 떼도 높이를 유지하며, 다시 잡아서 내리면 원래 위치로 돌아간다.
- 잡고 있는 동안에는 손이 코를 가려 얼굴 인식이 끊겨도 안경을 유지한다.
- 손 인식 로드에 실패해도 안경 착용 기능은 그대로 동작한다.

### 재질
| 부위 | 표현 |
|---|---|
| 테, 브릿지 | 광택 실버. 코드로 만든 스튜디오 환경맵을 반사한다 |
| 다리 | 브러시드 실버. 캔버스로 그린 결을 `roughnessMap`과 `bumpMap`에 사용한다 |
| 렌즈 | 투명한 바탕에 흰 에칭 선. 캔버스로 그린 `alphaMap`을 사용한다 |
| 표면 음영 | `toCreasedNormals`(40°)로 곡면은 매끄럽게, 모서리는 날카롭게 처리한다 |

## 사용 기술
빌드 도구와 프레임워크 없이 `index.html` 한 파일에서 ES 모듈로 실행한다. 라이브러리와 AI 모델은 실행 시 CDN에서 받으므로 인터넷 연결이 필요하다.

### 라이브러리
| 기술 | 버전 | 용도 |
|---|---|---|
| Three.js | 0.160.0 (r160) | 3D 렌더링 (`WebGLRenderer`, `PerspectiveCamera`, `MeshStandardMaterial`, `PMREMGenerator`, `CanvasTexture`) |
| Three.js `OBJLoader` | 0.160.0 | 안경 모델 로드 |
| Three.js `BufferGeometryUtils` | 0.160.0 | `toCreasedNormals`로 법선 계산 |
| MediaPipe Tasks Vision | 0.10.14 | 얼굴·손 인식 런타임 (WASM, `FilesetResolver`로 로드) |

### AI 모델
| 모델 | 파일 | 용도 |
|---|---|---|
| MediaPipe Face Landmarker | `face_landmarker.task` (float16) | 얼굴 랜드마크와 변환 행렬. 안경 위치, 크기, 회전, 귀 위 근사, 가림 메시에 사용 |
| MediaPipe Hand Landmarker | `hand_landmarker.task` (float16, 약 7.8MB) | 검지·엄지 끝 위치 인식 |

두 모델 모두 GPU delegate로 시작하고, 실패하면 CPU로 재시도한다.

### 브라우저 API
| 기술 | 용도 |
|---|---|
| WebGL | Three.js 렌더링 |
| `getUserMedia` | 전면 카메라 영상 |
| Canvas 2D | 다리 결, 렌즈 에칭 선 텍스처 생성 |
| ES Modules, Import Maps | 번들러 없이 CDN 모듈 로드 |
| CSS `object-fit`, `transform: scaleX(-1)` | 영상과 3D 캔버스를 같은 비율로 맞추고 좌우 반전 |

### 렌더링 기법
| 기법 | 구현 |
|---|---|
| 얼굴 가림 | 얼굴 메시를 `colorWrite: false`로 그려 깊이만 기록 |
| 금속 반사 | 코드로 만든 스튜디오 씬을 `PMREMGenerator`로 환경맵화 |
| 다리 재질 | `roughnessMap`, `bumpMap`, 박스 투영 UV |
| 렌즈 재질 | `alphaMap`, 평면 투영 UV |

### 개발·실행 도구
| 도구 | 용도 |
|---|---|
| Python `http.server` | 로컬 정적 서버 |
| Cloudflare Tunnel (`cloudflared`) | 모바일 테스트용 HTTPS 주소 |

## 3D 모델 정보
- OBJ 형식, 단위 mm, 오브젝트는 `frame`, `lens_L`, `lens_R`
- 전체 가로 약 130.7mm (다리 끝 기준), 앞면 가로 102.8mm, 세로 약 35.6mm, 다리 길이 약 154mm
- 렌즈 면이 +Z를 향한다.
- 브릿지는 렌즈 중간 높이에 있는 독립 부품이다 (x ±5mm).
- 다리는 힌지에서 내려가 z 약 -80mm에서 가장 낮게 꺾인 뒤, 다시 올라가 z 약 -130mm에서 가장 높아진다.

## 설정값 (`index.html`의 `CONFIG`)
| 키 | 기본값 | 설명 |
|---|---|---|
| `modelUrl` | `"glasses_v2.obj"` | 안경 모델 경로 |
| `frameRatio` | 1.4 | 안경 앞면 폭 ÷ 눈 바깥 모서리 거리 |
| `offsetY` | 0.0 | 세로 위치 보정 (앞면 폭 대비 비율, +면 위) |
| `smoothing` | 0.35 | 0~1. 클수록 빠르게 따라가고, 작을수록 떨림이 줄어든다 |
| `poseSource` | `"matrix"` | 회전 계산 방식 (`"matrix"` 또는 `"landmark"`) |
| `matrixRowMajor` | true | 변환 행렬의 행 우선 여부 |
| `yawSign`, `pitchSign` | 1 | `"landmark"` 방식의 좌우·상하 회전 방향 |
| `occlusion` | true | 얼굴 가림 on/off |
| `occluderBackMM` | 3 | 얼굴 점을 렌즈 평면 뒤로 미는 거리 |
| `occluderSideMarginMM` | 2 | 가림 메시 좌우 끝과 다리 안쪽 면의 간격 |
| `earTopOffsetMM` | 0 | 다리가 귀 위를 지나는 높이 보정 (+면 위) |
| `earOutMM` | 3 | 다리를 얼굴 윤곽 바깥으로 띄우는 거리 |
| `templeMaxAngleDeg` | 15 | 다리를 힌지에서 돌릴 수 있는 최대 각도 |
| `handLift` | true | 손으로 안경 옮기기 on/off |
| `handGrabRadius` | 0.18 | 브릿지를 잡았다고 보는 거리 (앞면 폭 대비 비율) |
| `handMaxLiftMM` | 10 | 안경을 올릴 수 있는 최대 높이 |
| `handDetectEvery` | 3 | 손 인식 실행 간격 (프레임). 클수록 가볍다 |

재질 밝기는 `makeStudioEnvironment`의 `panel` 강도, 렌즈 선 진하기는 `makeLensAlphaTexture`의 `globalAlpha`로 조절한다.

## 제약 사항
- 얼굴 1개, 손 1개만 인식한다.
- MediaPipe에 귀 랜드마크가 없어 얼굴 윤곽점(127, 356)으로 귀 위치를 근사한다. 차이가 나면 `earTopOffsetMM`으로 보정한다.
- 귀가 가림 메시에 없어서 귀 뒤로 넘어간 다리 끝은 가려지지 않는다.
- 정면에서는 다리 끝이 얼굴 옆쪽 위에 겹쳐 보일 수 있다.
- 안경이 항상 영상 위에 그려져서, 브릿지를 잡은 손가락이 안경 뒤에 있는 것처럼 보인다.
- 랜드마크 깊이 값은 추정치라, 고개를 많이 돌리면 크기나 가림 위치가 조금 어긋날 수 있다.
- 3D 모델 형상은 실제 제품 사진과 일부 다르다. 다리가 테의 세로 중간에서 시작하고, 다리 끝이 귀 뒤로 휘어 내려가지 않는다.

## 확인이 필요한 항목
- 재질과 손 기능의 실제 카메라 동작 (가짜 얼굴·손 데이터로만 확인함)
- 모바일 성능과 iOS 동작 (카메라 권한, GPU)
- 고개를 위아래로 움직일 때의 회전 방향. 반대로 보이면 `matrixRowMajor`와 회전 축 보정 상수 `FLIP`을 먼저 점검한다

## 다음 단계 제안
1. 실제 얼굴과 손으로 위 확인 항목 점검
2. 3D 모델 형상을 제품에 맞게 보강 (다리 시작 위치, 다리 끝 곡선)
3. 필요 시 OBJ를 GLB로 변환해 로딩 속도 개선
4. 귀와 손가락 가림 처리
