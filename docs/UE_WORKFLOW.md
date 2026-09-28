# Unreal Engine 워크플로우

리깅되지 않은 캐릭터 메시에 skintokens로 스켈레톤과 스킨 웨이트를 넣고,
Unreal Engine에 스켈레탈 메시로 가져오는 흐름이다.

> 상태: 아래 흐름은 아직 끝까지 검증하지 않았다. 단계마다 확인한 결과를 여기에 갱신한다.

## 전체 흐름

```
메시 준비 (AI 생성 / CC0 에셋)
  → glTF-Transform으로 정리
  → skintokens-cli skin  (마네킹 스켈레톤에 웨이트 입히기)
  → Unreal 임포트 (기존 마네킹 스켈레톤 지정)
  → 마네킹 애니메이션을 리타깃 없이 재생
```

## 1. 빌드 (Windows는 WSL)

업스트림 빌드 문서는 Linux 기준이다. Windows에서는 WSL(Ubuntu)에서 빌드한다.
MSVC 네이티브 빌드는 확인하지 않았다.

```sh
sudo apt install git cmake ninja-build g++ nlohmann-json3-dev \
  libvulkan-dev glslc
git submodule update --init --recursive
cmake -S . -B build/release -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build/release -j
```

WSL에서 GPU(Vulkan)를 못 잡으면 `-DSKINTOKENS_ENABLE_VULKAN=OFF`로 빌드하고
`--device cpu`로 돌린다. 느리지만 동작은 같다.

가중치:

```sh
hf download LocalAI-io/SkinTokens-GGUF --include "F16/*" \
  --local-dir models/SkinTokens-GGUF
```

## 2. 메시 준비

- 입력은 삼각형 메시 `.glb` 하나. 사람형이고 T/A 포즈에 가까울수록 결과가 좋다.
- 소스 예: Hunyuan3D, Rodin 같은 AI 생성 모델, Quaternius, Kenney (CC0)
- [glTF-Transform](https://github.com/donmccurdy/glTF-Transform)으로 정리한다.
  Unreal 임포터가 못 읽을 수 있으므로 Draco/meshopt 압축은 쓰지 않는다.

```sh
npx @gltf-transform/cli weld in.glb welded.glb
npx @gltf-transform/cli simplify welded.glb character.glb --ratio 0.5
```

## 3. 마네킹 스켈레톤 GLB 만들기

리타깃 없이 기존 애니메이션을 쓰려면 Unreal 마네킹 스켈레톤에 웨이트를 입힌다.

1. Unreal에서 마네킹 스켈레탈 메시(`SKM_Manny` 등)를 FBX로 익스포트
2. Blender로 FBX를 열고 glTF 2.0(`.glb`)으로 익스포트
3. 스케일 확인: Unreal은 cm, glTF는 m 단위다. Blender 익스포트에서 단위를 맞춘다.

## 4. 스킨 웨이트 생성

```sh
./build/release/bin/skintokens-cli skin \
  models/SkinTokens-GGUF/F16 \
  character.glb manny-skeleton.glb character-rigged.glb \
  --device vulkan --fit global
```

- `--fit global`: 스켈레톤을 메시 높이와 중심에 맞게 균일 스케일·이동한다. 뼈 비율은 유지된다.
- 팔이 메시 밖으로 나가면 `--fit articulated`(실험적)를 시도한다.
- `rig` 명령(스켈레톤 자동 생성)은 업스트림도 실험적이라고 한다. 게임용은 `skin`을 쓴다.

## 5. Unreal 임포트

1. `character-rigged.glb`를 Content Browser로 드래그
2. 임포트 옵션에서 Skeleton을 기존 마네킹 스켈레톤(`SK_Mannequin`)으로 지정
3. 마네킹 애니메이션을 적용해 변형을 확인
4. 어깨, 겨드랑이, 골반의 웨이트는 대부분 손으로 다듬어야 한다 (Blender 웨이트 페인트)

## 주의

- 코드는 Apache-2.0. 모델 가중치는 원본
  [VAST-AI SkinTokens](https://github.com/VAST-AI-Research/SkinTokens)의 라이선스를
  따로 확인한 뒤 상업적으로 쓴다.
- 학습 분포와 다른 형태(동물, 로봇, 비정형)는 결과가 불안정하다.
