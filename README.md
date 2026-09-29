# Vision-Nut-Picking-Cobot

사용자의 직업과 포만감을 음성으로 입력받아 맞춤 견과류를 추천하고, Doosan M0609 협동로봇이 Vision 기반으로 견과류를 Pick-and-Place한 뒤 컨베이어를 통해 전달하는 ROS 2 기반 자동화 시스템입니다.

## 담당 역할 및 주요 기여

- 견과류 탐지를 위한 YOLO 기반 OBB 객체 탐지 모델 개발에 참여하여 데이터 수집·라벨링, Polygon 기반 재라벨링, 반복 학습 및 모델 성능 비교 실험을 수행했습니다.
- Python 음성 추천 파이프라인에서 LangChain의 `ChatOpenAI`를 통해 OpenAI GPT-4o API를 연동하고, 사용자의 자연어 입력에서 직업·포만감을 분석해 견과류 종류·수량 및 추천 근거를 생성하는 로직을 구현했습니다.
- 단순 음성 명령을 넘어 STT → LLM 분석 → 후속 질문 → 사용자 응답 → 추천 → TTS/UI로 이어지는 양방향 상호작용 흐름을 설계·개선했습니다.
- ROS2·YOLO Object Detection용 Docker 실행환경을 구축하고, 음성·로봇 제어 등 OD와 무관한 의존성을 제외한 전용 컨테이너 구조로 재구성했습니다.

**Technical Deep Dive:** [Notion](https://capable-moss-bbd.notion.site/Technical-Deep-Dive-2-3e046f508ab380f5bf03dc062896cfbc?pvs=74)

---

## 동작 개요

1. 사용자가 웹 UI(또는 CLI)에서 음성 세션을 시작한다.
2. Wake word 감지 후 시스템이 사용자에게 **직업**과 **현재 포만감**을 TTS로 질문한다.
3. STT(Whisper)가 사용자의 답변을 전사하고, LLM이 자연어 입력에서 직업 특성과 포만감 수준을 분석한다.
4. 분석 결과를 기반으로 견과류 종류와 수량을 결정하고, 추천 결과와 세션 상태를 Supabase에 기록한다.
5. `task_manager_node`가 주문을 읽어, 견과류별로 perception → pick → place 사이클을 실행한다.
6. `robot_control_node`가 픽 시퀀스(approach → grasp → verify_grip → lift → transit → place → retreat → home)를 수행한다.
7. 컨베이어가 `place_ready` 신호의 엣지에서 한 단위 전진한다.
8. 주문 완료 시 상태가 백엔드에 미러링되어 웹 UI가 "completed"를 표시한다.

## 패키지 구성

| 패키지 | 역할 |
|---|---|
| `cobot_bringup` | 통합 launch 파일 (`full_system`, `bringup_supabase`, `perception`, `robot`, `host_system`) |
| `cobot_config` | 워크스페이스/캘리브레이션 공용 설정 |
| `cobot_db` | DB 어댑터 (Firestore / Supabase) |
| `cobot_msgs` | 커스텀 ROS 메시지/액션 정의 |
| `cobot_object_detection` | YOLO 기반 견과류 검출 |
| `cobot_perception` | 카메라 ↔ 로봇 좌표 변환 및 perception 트리거 |
| `cobot_robot_control` | Doosan 로봇 + RG2 그리퍼 제어 (액션 서버) |
| `cobot_task_manager` | 주문 단위 task 오케스트레이션, verification/correction 루프 |
| `cobot_voice` | 음성 파이프라인 (wake word, STT, TTS, 키워드 추출, 룰 엔진) |
| `conveyor_controller` | Arduino stepper 벨트 제어 노드 |
| `nuts_data_recording` | 견과류 캘리브레이션/데이터 수집용 레코더 |
| `web_stt_firebase[_v2]` / `web_stt_supabase_v2` | 웹 UI (React + Vite + Three.js) |

## 빠른 실행

> 워크스페이스 경로는 `~/cobot_ws` 기준 (`src/cobot2`에 본 repo가 위치). 상세 사전 점검과 환경 변수는 `docs/03_run_manual.md` 참조.

```bash
cd ~/cobot_ws
colcon build --symlink-install
source install/setup.bash

# Supabase 경로 (현재 기본). bringup_supabase는 full_system의 Supabase 프리셋 wrapper.
ros2 launch cobot_bringup bringup_supabase.launch.py

# Firestore 경로 (레거시 — Supabase 마이그레이션 이후 사용 안 함, 필요 시에만)
ros2 launch cobot_bringup full_system.launch.py \
    enable_firebase_status_bridge:=true enable_supabase_status_bridge:=false

# 웹 UI는 rosbridge_websocket을 거쳐 ROS와 통신한다.
# 별도 터미널에서:
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
cd src/cobot2/web_stt_supabase_v2 && npm run dev
```

## 문서

| 문서 | 내용 |
|---|---|
| [`docs/01_system_architecture.md`](docs/01_system_architecture.md) | 시스템 구성, 데이터 계약, 하드웨어 아키텍처, 안전 설계 |
| [`docs/02_ros_node_architecture.md`](docs/02_ros_node_architecture.md) | 노드별 ROS 인터페이스 레퍼런스 |
| [`docs/03_run_manual.md`](docs/03_run_manual.md) | 단계별 운영자 실행 매뉴얼 (Firestore 경로) |
| [`docs/04_validation_checklist.md`](docs/04_validation_checklist.md) | 사전 점검/수용 테스트 체크리스트 |
| [`docs/05_clustered_nuts_handling.md`](docs/05_clustered_nuts_handling.md) | 클러스터된 견과류 처리 정책 |
| [`docs/06_perception_trigger_redesign.md`](docs/06_perception_trigger_redesign.md) | Perception 트리거 재설계 |
| [`docs/07_verification_and_correction_loop.md`](docs/07_verification_and_correction_loop.md) | Verification / correction 루프 |
| [`docs/08_cluster_handling_implementation.md`](docs/08_cluster_handling_implementation.md) | 클러스터 처리 구현 노트 |
| [`docs/09_supabase_migration.md`](docs/09_supabase_migration.md) | Supabase 백엔드 경로 (Firestore 대체) |
| [`docs/cleanup_deletion_proposal.md`](docs/cleanup_deletion_proposal.md) | 아카이브 및 삭제 계획 |

## 하드웨어

- Doosan Robotics M0609 (6-DOF)
- OnRobot RG2 그리퍼
- Intel RealSense D-시리즈 카메라
- Arduino 기반 stepper 컨베이어

## 아카이브 안내

`_archive_cleanup/<YYYYMMDD>/` 디렉토리와 `docs/_archive/`는 **활성 코드/문서가 아니다**. 이력 보존 목적으로만 남아 있으며, `colcon build`, `source`, `import`, 실행 대상 모두 제외 대상이다. 자세한 사유는 `docs/cleanup_deletion_proposal.md` 및 각 배치의 `cleanup_manifest.md` 참조.
