# PRD — 2D Character Animation Skill

**Project Working Name:** `sprite-animation-forge`  
**Type:** AI Agent Skill  
**Version:** 0.1  
**Status:** Draft  
**Primary Target:** Codex / Claude Code / Agent Skills compatible runtime  
**Primary Use Case:** 2D 게임 캐릭터 애니메이션 및 Sprite Sheet 생성

---

# 1. 제품 개요

## 1.1 목적

사용자가 캐릭터 이미지 또는 자연어 설명을 제공하면 AI Agent가 다음 과정을 자동으로 수행하는 2D 게임 캐릭터 애니메이션 생성 Skill을 개발한다.

1. 캐릭터 분석
2. 애니메이션 요구사항 추론
3. 액션별 프레임 구성 결정
4. 이미지 생성
5. 프레임 분리
6. 배경 제거
7. 캐릭터 크기 및 위치 정규화
8. 품질 검사(QC)
9. 필요 시 재생성 또는 재처리
10. Sprite Sheet / 개별 PNG / GIF / Atlas 생성
11. 게임 엔진용 메타데이터 출력

기본 철학은 다음과 같다.

> AI는 그림을 만들고, 코드는 그림을 정렬·분리·검증·패키징한다.

Creative sprite art 자체를 Python/PIL/SVG 등으로 생성하지 않고 이미지 생성 모델을 사용하며, 후처리는 deterministic script가 담당한다.

---

# 2. 배경

`agent-sprite-forge/generate2dsprite`는 자연어 요청에서 asset type, action, view, sheet, frame count, anchor, scale strategy 등을 추론하고 이미지 생성 → chroma-key 처리 → frame extraction → alignment → QC → export 파이프라인을 제공한다.

특히 플레이어 캐릭터처럼 일관성이 중요한 대상은 여러 액션을 하나의 거대한 이미지로 생성하지 않고,

`Idle → QC`

`Walk → QC`

`Run → QC`

`Attack → QC`

형태로 각각 생성한 뒤 마지막 단계에서 Atlas를 조립하는 방식을 사용한다.

본 프로젝트는 이 구조를 기본 원칙으로 채택하면서 캐릭터 애니메이션 제작에 특화된 기능을 추가한다.

---

# 3. 해결하려는 문제

AI 이미지 생성 모델에 단순히

> 이 캐릭터의 16프레임 sprite sheet를 만들어줘.

라고 요청하면 다음 문제가 자주 발생한다.

- 프레임마다 얼굴이 달라짐
- 의상이나 액세서리가 변경됨
- 캐릭터 크기가 프레임마다 달라짐
- 발 위치가 흔들림
- 카메라 줌이 변화함
- 프레임 경계를 캐릭터가 침범함
- 무기나 이펙트 때문에 캐릭터가 작아짐
- 배경 제거가 깨끗하지 않음
- walk cycle 순서가 자연스럽지 않음
- 서로 다른 액션이 섞임
- 실제 게임 엔진에서 바로 사용하기 어려움

본 Skill은 이미지 생성 자체보다 **생성 파이프라인과 일관성 관리**를 통해 이 문제를 해결한다.

---

# 4. 제품 목표

## 핵심 목표

사용자가 다음 정도의 요청만 해도 결과물을 생성할 수 있어야 한다.

> 이 캐릭터로 게임용 애니메이션 만들어줘.  
> idle, walk, run, attack이 필요해.

Agent가 나머지를 판단한다.

예:

- idle → 4 frames
- walk → 6 frames
- run → 6 frames
- attack → 6 frames
- anchor → feet
- view → side
- transparent background
- Phaser atlas export

사용자가 모든 기술적 옵션을 지정할 필요가 없어야 한다.

---

# 5. 제품 범위

## 5.1 MVP 범위

MVP에서는 **캐릭터 애니메이션**을 최우선으로 지원한다.

지원 Asset:

- player
- character
- npc
- creature
- enemy

지원 Action:

- idle
- walk
- run
- attack
- shoot
- cast
- jump
- fall
- hurt
- death

추후 지원:

- dodge
- roll
- block
- parry
- climb
- swim
- fly
- hover
- charge
- transform
- emote
- custom action

---

# 6. 지원 View

```text
side
topdown
3/4
front
rear
```

MVP 우선순위:

1. side
2. topdown
3. 3/4

---

# 7. 핵심 사용자 경험

## Case A — 기존 캐릭터 이미지

사용자:

> 이 이미지 캐릭터로 idle, walk, run, attack을 만들어줘.

시스템:

```text
Reference Analysis
      ↓
Character Identity Profile
      ↓
Animation Plan
      ↓
Idle Generation
      ↓
QC
      ↓
Walk Generation
      ↓
QC
      ↓
Run Generation
      ↓
QC
      ↓
Attack Generation
      ↓
QC
      ↓
Normalize
      ↓
Atlas Assembly
      ↓
Export
```

---

## Case B — 텍스트에서 생성

사용자:

> 빨간 후드와 검을 든 2D 횡스크롤 기사 캐릭터를 만들고 애니메이션도 만들어줘.

시스템:

```text
Character Design
       ↓
Canonical Reference
       ↓
Identity Profile
       ↓
Idle / Walk / Run / Attack
       ↓
QC
       ↓
Export
```

---

# 8. Character Identity Lock

본 Skill의 가장 중요한 기능이다.

Reference 이미지에서 다음 요소를 추출한다.

```yaml
identity:
  silhouette:
  body_ratio:
  head_ratio:
  hair:
  face:
  eyes:
  clothing:
  primary_colors:
  secondary_colors:
  weapon:
  accessories:
  outline_style:
  shading_style:
  camera_angle:
  orientation:
```

모든 액션 프롬프트에서 이 정보를 반복적으로 유지한다.

변경 가능한 것은 기본적으로 다음뿐이다.

```text
pose
limb position
body lean
facial expression
weapon motion
secondary motion
```

캐릭터 자체 디자인은 변경하지 않는다.

Reference가 있을 경우 silhouette, palette, 얼굴 특징, 의상, 주요 액세서리를 유지하면서 요청된 액션만 변화시키는 방식은 `generate2dsprite`에서도 핵심 규칙으로 사용된다.

---

# 9. Animation Planning Engine

Agent가 사용자 요청을 분석하여 자동으로 Animation Plan을 생성한다.

예:

```json
{
  "character": "hero",
  "view": "side",
  "anchor": "feet",
  "actions": {
    "idle": {
      "frames": 4,
      "grid": "2x2",
      "loop": true,
      "fps": 6
    },
    "walk": {
      "frames": 6,
      "grid": "2x3",
      "loop": true,
      "fps": 10
    },
    "run": {
      "frames": 6,
      "grid": "2x3",
      "loop": true,
      "fps": 12
    },
    "attack": {
      "frames": 6,
      "grid": "2x3",
      "loop": false,
      "fps": 12
    }
  }
}
```

---

# 10. 기본 Frame Preset

| Action | Default Frames | Grid | Loop |
|---|---:|---|---|
| idle | 4 | 2×2 | Yes |
| walk | 6 | 2×3 | Yes |
| run | 6 | 2×3 | Yes |
| attack | 6 | 2×3 | No |
| shoot | 4 | 2×2 | No |
| cast | 6 | 2×3 | No |
| jump | 4 | 2×2 | No |
| hurt | 4 | 2×2 | No |
| death | 8 | 2×4 | No |

사용자가 frames를 지정하면 사용자 값을 우선한다.

원본 `generate2dsprite` 역시 4 frames → `2x2`, 6 → `2x3`, 8 → `2x4`처럼 긴 single-row 대신 compact multi-row grid를 기본으로 사용한다. 긴 single-row 생성이 캐릭터 위치 드리프트와 crop 문제를 일으키기 때문이다.

---

# 11. Action 단위 생성

다음과 같은 Mixed Action Sheet를 직접 생성하지 않는다.

```text
┌──────┬──────┬──────┬──────┐
│ Idle │ Idle │ Idle │ Idle │
├──────┼──────┼──────┼──────┤
│ Walk │ Walk │ Walk │ Walk │
├──────┼──────┼──────┼──────┤
│ Run  │ Run  │ Run  │ Run  │
├──────┼──────┼──────┼──────┤
│Attack│Attack│Attack│Attack│
└──────┴──────┴──────┴──────┘
```

대신:

```text
idle_raw.png
walk_raw.png
run_raw.png
attack_raw.png
```

를 각각 생성한다.

각각 QC에 통과하면 마지막에:

```text
character-atlas.png
character-atlas.json
```

을 생성한다.

원본 Skill 역시 메인 캐릭터의 mixed-action atlas를 직접 생성하지 않고 action별 생성 및 QC 후 deterministic assembly를 요구한다.

---

# 12. Prompt Generation

Agent는 내부적으로 Action Prompt를 생성한다.

구조:

```text
REFERENCE IDENTITY
+
ART STYLE
+
CAMERA
+
ACTION
+
FRAME SEQUENCE
+
CONSISTENCY RULES
+
GRID RULES
+
BACKGROUND RULE
```

예:

```text
Create a 2x3 sprite animation grid.

Same character as the provided reference.

Preserve exactly:
- silhouette
- body proportions
- hairstyle
- facial features
- red jacket
- black pants
- sword design
- color palette

Side-view run animation facing right.

6 sequential animation frames.

Maintain:
- identical character scale
- identical camera distance
- stable feet baseline
- same facing direction
- full body visible
- character centered in every cell

No frame may overlap another cell.

Background must be flat #FF00FF.
No shadows outside the character.
No text.
```

---

# 13. Background 처리

기본 방식:

```text
#FF00FF
```

Solid Magenta background를 사용한다.

Pipeline:

```text
Generated Image
       ↓
Color Distance Detection
       ↓
Chroma Key Removal
       ↓
Edge Cleanup
       ↓
Alpha PNG
```

허용 tolerance를 설정한다.

예:

```yaml
background:
  mode: chroma
  color: "#FF00FF"
  tolerance: 20
  edge_cleanup: true
```

향후:

```text
native transparency
background segmentation
AI mask
```

를 지원한다.

---

# 14. Frame Extraction

생성 이미지에서 grid를 기준으로 프레임을 분리한다.

```text
raw_sheet.png

↓

frame_000.png
frame_001.png
frame_002.png
frame_003.png
...
```

지원 grid:

```text
2x2
2x3
2x4
3x3
3x4
4x4
custom
```

---

# 15. Component Detection

각 프레임의 foreground object를 분석한다.

지원 모드:

```text
largest
all
```

### largest

캐릭터 body에 사용.

작은 stray pixel 또는 분리된 noise는 제거한다.

### all

다음에 사용한다.

- magic FX
- projectile
- explosion
- impact
- detached particles

---

# 16. Alignment

지원 Anchor:

```text
center
bottom
feet
```

캐릭터 기본값:

```text
feet
```

정규화:

```text
Frame 1 feet Y = 112
Frame 2 feet Y = 112
Frame 3 feet Y = 112
Frame 4 feet Y = 112
```

X축 중심 역시 일정 범위 내에서 유지한다.

---

# 17. Scale Normalization

두 가지 전략을 제공한다.

## fit

캐릭터 bounding box를 기준으로 target cell에 맞춘다.

적합:

- idle
- walk
- run
- NPC
- projectile
- FX

## preserve

생성된 캐릭터 scale을 최대한 유지한다.

적합:

- sword attack
- spear attack
- 긴 망토
- 큰 무기
- wide pose

원본 Skill에서도 긴 무기나 넓은 melee pose로 인해 bbox-fit 과정에서 캐릭터 자체가 작아지는 문제를 방지하기 위해 `preserve + feet alignment` 전략을 사용한다.

---

# 18. Character Scale Profile

첫 번째 승인된 animation을 기준으로 캐릭터 스케일을 저장한다.

예:

```json
{
  "character": "hero",
  "reference_action": "idle",
  "body_height": 104,
  "body_width": 62,
  "feet_y": 118,
  "center_x": 64,
  "target_cell": [128,128]
}
```

이후:

```text
walk
run
attack
jump
hurt
```

생성 시 이 값을 기준으로 QC한다.

---

# 19. 자동 Quality Control

모든 animation sheet에 대해 자동 QC를 수행한다.

## QC-01 Cell Boundary

캐릭터가 cell 경계를 침범하지 않아야 한다.

```text
edge_touch = false
```

---

## QC-02 Scale Drift

프레임간 캐릭터 크기 편차:

```text
≤ 10%
```

권장:

```text
≤ 5%
```

---

## QC-03 Anchor Drift

feet anchor variation:

```text
≤ 3 px
```

---

## QC-04 Center Drift

캐릭터 중심 X 편차:

```text
≤ configurable threshold
```

---

## QC-05 Empty Frame

캐릭터가 없는 프레임을 검출한다.

---

## QC-06 Duplicate Frame

perceptual hash 등을 사용하여 거의 동일한 frame을 탐지한다.

---

## QC-07 Character Scale

idle을 기준으로 다른 body action이 지나치게 작아지면 실패 처리한다.

예:

```text
attack character height
<
idle height × 0.85
```

---

## QC-08 Reference Consistency

향후 Vision Model을 이용해 다음 항목의 consistency score를 계산한다.

```text
costume
hair
face
palette
weapon
silhouette
```

---

# 20. QC 결과

`qc-report.json`

예:

```json
{
  "action": "run",
  "status": "pass",
  "frames": 6,
  "checks": {
    "edge_touch": false,
    "scale_variance": 0.042,
    "anchor_variance": 1.8,
    "duplicate_frames": [],
    "empty_frames": []
  }
}
```

---

# 21. 자동 복구

QC 실패 시 Agent가 원인을 분석한다.

### 경계 침범

```text
Regenerate
+
increase safe margin
```

### 캐릭터 크기 불일치

```text
Reprocess
+
shared_scale
```

또는:

```text
Regenerate
+
stronger scale constraint
```

### Attack에서 캐릭터가 작음

```text
switch:
fit → preserve
```

### FX 때문에 bbox가 커짐

```text
split:
body
+
fx
```

---

# 22. Body와 FX 분리

Attack 등의 경우:

```text
attack-body.png
slash-fx.png
impact-fx.png
```

로 분리할 수 있다.

게임에서는:

```text
Character Sprite
      +
Slash FX Sprite
      +
Impact Sprite
```

로 조합한다.

원본 Skill도 controllable hero의 attack/shoot/cast에서 큰 slash arc, muzzle flash, projectile, impact 등을 body sheet와 기본적으로 분리한다.

---

# 23. Output Structure

```text
sprites/
└── hero/
    ├── reference/
    │   └── character.png
    │
    ├── identity/
    │   └── character-profile.json
    │
    ├── idle/
    │   ├── raw.png
    │   ├── clean.png
    │   ├── sheet.png
    │   ├── animation.gif
    │   ├── qc-report.json
    │   └── frames/
    │       ├── 000.png
    │       ├── 001.png
    │       ├── 002.png
    │       └── 003.png
    │
    ├── walk/
    ├── run/
    ├── attack/
    │
    ├── fx/
    │
    ├── atlas/
    │   ├── hero.png
    │   ├── hero.json
    │   └── hero.meta.json
    │
    ├── character-scale-profile.json
    └── manifest.json
```

원본 `generate2dsprite` 역시 raw sheet, cleaned sheet, transparent sheet, 개별 frame PNG, GIF, prompt, pipeline metadata 등을 결과물로 제공한다.

---

# 24. Export Formats

MVP:

```text
PNG Frames
PNG Sprite Sheet
Animated GIF
JSON Metadata
```

---

# 25. Game Engine Export

## Generic

```text
sprite.png
sprite.json
```

## Phaser

우선 지원 대상.

```json
{
  "frames": {
    "idle_0": {},
    "idle_1": {},
    "walk_0": {},
    "walk_1": {}
  }
}
```

추가 파일:

```text
animations.json
```

예:

```json
{
  "idle": {
    "frames": [
      "idle_0",
      "idle_1",
      "idle_2",
      "idle_3"
    ],
    "frameRate": 6,
    "repeat": -1
  }
}
```

향후:

- Godot
- Unity
- PixiJS
- Cocos
- Aseprite compatible metadata

---

# 26. Skill Parameters

```yaml
asset_type:
  - player
  - npc
  - character
  - creature
  - enemy

action:
  - idle
  - walk
  - run
  - attack
  - shoot
  - cast
  - jump
  - fall
  - hurt
  - death
  - custom

view:
  - side
  - topdown
  - 3/4
  - front
  - rear

frames:
  - auto
  - number

grid:
  - auto
  - 2x2
  - 2x3
  - 2x4
  - 3x3
  - 3x4
  - 4x4
  - custom

anchor:
  - center
  - bottom
  - feet

scale_strategy:
  - fit
  - preserve

art_style:
  - auto
  - pixel_art
  - retro_pixel
  - pixel_inspired
  - clean_hd
  - project_native

reference:
  - none
  - attached_image
  - generated_image
  - local_file

engine:
  - generic
  - phaser
  - godot
  - unity
```

---

# 27. Skill 사용 예

### 기본

```text
Use $sprite-animation-forge to create idle, walk and run animations
for this character.
```

### 한국어

```text
$sprite-animation-forge를 사용해서
이 캐릭터의 idle 4프레임,
walk 6프레임,
run 6프레임,
attack 6프레임을 만들어줘.
```

### Phaser

```text
이 캐릭터의 게임용 애니메이션을 만들고
Phaser atlas와 animations.json까지 생성해줘.
```

### 자동 판단

```text
이 캐릭터를 횡스크롤 액션 게임에서 사용할 수 있게 만들어줘.
```

Agent가 자동으로:

```text
idle
walk
run
jump
attack
hurt
death
```

등의 필요 animation bundle을 제안 또는 생성한다.

---

# 28. 내부 Architecture

```text
                ┌────────────────────┐
                │    User Request    │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ Intent / Planner   │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ Identity Analyzer  │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ Animation Planner  │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ Prompt Generator   │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ Image Generator    │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ Sprite Processor   │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ QC Engine          │
                └───────┬─────┬──────┘
                        │     │
                     PASS    FAIL
                        │     │
                        │     └──→ Regenerate
                        ↓
                ┌────────────────────┐
                │ Atlas Builder      │
                └─────────┬──────────┘
                          ↓
                ┌────────────────────┐
                │ Engine Exporter    │
                └────────────────────┘
```

---

# 29. Skill Repository 구조

권장:

```text
sprite-animation-forge/
├── SKILL.md
│
├── references/
│   ├── animation-rules.md
│   ├── prompt-rules.md
│   ├── character-consistency.md
│   ├── qc-rules.md
│   ├── phaser-export.md
│   └── examples.md
│
├── scripts/
│   ├── process_sprite.py
│   ├── split_frames.py
│   ├── remove_background.py
│   ├── align_frames.py
│   ├── normalize_scale.py
│   ├── qc_frames.py
│   ├── build_atlas.py
│   └── export_phaser.py
│
├── schemas/
│   ├── animation-plan.schema.json
│   ├── character-profile.schema.json
│   └── manifest.schema.json
│
└── tests/
    ├── fixtures/
    ├── test_split_frames.py
    ├── test_alignment.py
    ├── test_qc.py
    └── test_atlas.py
```

---

# 30. 구현 기술

## Agent layer

```text
SKILL.md
Markdown
JSON Schema
```

## Image Processing

```text
Python 3.11+
Pillow
NumPy
```

선택:

```text
OpenCV
imagehash
scikit-image
```

## Image Generation

특정 provider에 종속되지 않는 abstraction을 목표로 한다.

예:

```text
image_gen
GPT Image
Gemini Image
Flux
other supported image tool
```

Skill은 runtime에 존재하는 image-generation capability를 사용한다.

---

# 31. 비기능 요구사항

## Deterministic Postprocessing

동일한 raw image + 동일 옵션이면 항상 동일 결과가 생성되어야 한다.

---

## Provider Independence

Image provider 특성은 adapter 또는 prompt profile로 분리한다.

---

## Reproducibility

다음을 기록한다.

```text
prompt
reference
parameters
processor settings
generation model
output hashes
QC result
```

---

## Non-destructive

원본 생성 이미지는 삭제하거나 덮어쓰지 않는다.

---

# 32. MVP 성공 기준

다음 시나리오를 통과하면 MVP 완료로 판단한다.

### Scenario 1

Reference character → idle 4 frames

결과:

- 4 PNG
- transparent sprite sheet
- GIF
- QC JSON

### Scenario 2

Reference character → walk 6 frames

결과:

- 동일 캐릭터 유지
- feet alignment 유지
- scale drift ≤ 10%
- cell boundary violation 없음

### Scenario 3

Hero bundle

```text
idle
walk
run
attack
```

생성 후 하나의 Phaser Atlas 생성.

### Scenario 4

Attack에 긴 sword가 포함되어도 캐릭터 크기가 idle 대비 지나치게 작아지지 않는다.

### Scenario 5

QC 실패 시 Agent가 실패 이유를 식별하고 reprocess 또는 regenerate 전략을 선택한다.

---

# 33. Phase 2

MVP 이후:

### Character Consistency Scoring

Vision model을 이용해:

```text
Identity score: 0-100
```

계산.

---

### Automatic Best-of-N

한 action을 여러 번 생성:

```text
Candidate A
Candidate B
Candidate C
```

QC score가 가장 높은 결과 선택.

---

### Direction Generation

```text
left
right
up
down
```

지원.

특히:

```text
topdown 4-direction walk
```

자동 생성.

---

### Animation Semantic QC

실제 pose sequence를 분석해:

```text
walk cycle continuity
run cycle continuity
attack anticipation
attack impact
recovery
```

검증.

---

# 34. Phase 3

## Video → Sprite

```text
Reference
   ↓
AI Video
   ↓
Frame Extraction
   ↓
Frame Selection
   ↓
Normalization
   ↓
Sprite Sheet
```

고프레임 애니메이션 생성용 보조 pipeline으로 추가한다.

이는 기본 Sprite 생성 경로를 대체하지 않는다.

---

# 35. 명시적 Non-Goals

MVP에서는 다음을 목표로 하지 않는다.

- 완전한 2D 게임 제작
- collision 자동 생성
- 게임 플레이 코드 생성
- 3D 캐릭터
- skeletal animation
- Spine rig 자동 생성
- AI video 기반 animation
- tilemap 제작
- map 생성

Skill의 책임은:

> **게임에서 바로 사용할 수 있는 일관성 높은 2D 캐릭터 animation asset을 생성하는 것**

까지로 제한한다.

---

# 36. Upstream Compatibility 정책

`agent-sprite-forge/generate2dsprite`의 다음 개념은 핵심 compatibility 대상으로 한다.

- 자연어 기반 asset planning
- action inference
- compact grid generation
- chroma-key background
- frame extraction
- shared scaling
- anchor alignment
- component selection
- edge-touch QC
- transparent output
- GIF preview
- action별 생성
- body / FX separation
- final atlas assembly

다만 내부 구현은 모듈화하여 확장한다.

원본 processor는 chroma cleanup, frame splitting, component filtering, scaling, alignment, QC metadata 및 transparent/GIF export를 deterministic 후처리 역할로 제한하고 있다. 본 프로젝트도 같은 역할 분리를 유지한다.

---

# 37. 라이선스 및 구현 정책

Upstream `agent-sprite-forge`는 MIT License로 공개되어 있으며 사용, 수정, 배포 등이 허용되지만 저작권 고지와 라이선스 고지를 유지해야 한다.

따라서 구현 방식은 두 가지가 가능하다.

### Option A — Reimplementation

동작 원리와 아이디어만 참고하고 코드를 새로 작성한다.

권장 방식.

### Option B — MIT Code Reuse

검증된 processor 일부를 가져와 개선한다.

이 경우 원본 copyright 및 MIT License notice를 저장소에 유지한다.

---

# 38. 제품 핵심 차별점

본 Skill은 단순한:

```text
Prompt → Sprite Sheet
```

도구가 아니다.

최종 구조는:

```text
Reference
   ↓
Identity Lock
   ↓
Animation Planning
   ↓
Action-by-Action Generation
   ↓
Frame Extraction
   ↓
Alignment
   ↓
Scale Normalization
   ↓
Automated QC
   ↓
Regeneration / Reprocessing
   ↓
Engine Atlas
```

이어야 한다.

특히 다음 세 요소를 제품의 핵심 가치로 정의한다.

**1. Character Identity Lock**

여러 animation에서도 동일 캐릭터를 유지한다.

**2. Automated Sprite QC**

생성된 그림을 그대로 사용하지 않고 실제 게임 asset으로 사용할 수 있는지 검사한다.

**3. Game-ready Export**

단순 PNG 생성으로 끝나지 않고 Phaser 등의 게임 엔진에서 즉시 사용할 수 있는 형태까지 생성한다.

---

# 39. 최종 MVP 정의

사용자가 캐릭터 이미지 하나와 다음 요청을 제공한다고 가정한다.

> 이 캐릭터를 횡스크롤 게임에서 사용할 수 있도록 idle, walk, run, attack 애니메이션을 만들어줘.

Skill이 별도 추가 질문 없이 가능한 범위에서 판단하여 최종적으로 다음을 만들어야 한다.

```text
hero/
├── idle/
├── walk/
├── run/
├── attack/
├── preview/
│   ├── idle.gif
│   ├── walk.gif
│   ├── run.gif
│   └── attack.gif
├── atlas/
│   ├── hero.png
│   └── hero.json
├── animations.json
├── character-profile.json
├── character-scale-profile.json
├── qc-report.json
└── manifest.json
```

이 결과물을 **Phaser 프로젝트의 assets 폴더에 복사하여 바로 사용할 수 있는 수준**으로 만드는 것을 MVP의 최종 목표로 한다.