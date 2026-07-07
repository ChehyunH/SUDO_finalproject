# SUDO (Smart Unmanned Daily Outlet)

> 고객 공간과 로봇 작업 공간을 분리한 무인 픽앤플레이스 판매 시스템 — 손님은 진열대에 손대지 않고 주문만 하면, 로봇 팔이 상품을 집어 락커에 넣고 손님은 QR로 수령한다.

---

## 1. 실제 시스템 구성

별도의 중앙 "Main 노드"는 없다. **키오스크 웹 백엔드가 주문 큐·완료 판정·재고/락커 관리를 겸한다.**

| 컴포넌트 | 코드 | 역할 |
|----------|------|------|
| 키오스크 웹 | `web_kiosk/` (React+Vite / FastAPI, :8000) | 손님 주문 화면 + QR 수령 |
| 키오스크 백엔드 (사실상의 두뇌) | `web_kiosk/backend/main.py` (`KioskBackend`) | 주문 큐 → 로봇 순차 투입, 완료 판정, 재고·락커 |
| 검출 노드 (눈) | `dsr_realsense_pick_place/object_detector.py` | YOLOv8 + FastSAM 인식 → `/detected_objects` |
| 픽앤플레이스 노드 (팔) | `dsr_realsense_pick_place/pick_place_node.py` | 실제 집기·이송 FSM |
| 그리퍼 브릿지 | `dsr_gripper_tcp/` | 두산 TCP/DRL로 집게 제어 |
| 초음파 노드 | `dsr_realsense_pick_place/ultrasonic_node.py` + 아두이노 | 파지 시 거리 측정 |
| 저장소 | `dsr_realsense_pick_place/task_repository.py` | 주문/큐/락커=JSON, 재고/통계/이력=SQLite (하이브리드) |
| 관리자/로봇제어 웹 | `/admin`, `web_control_node.py` (:8080) | 재고·주문 관리 / 로봇·그리퍼 수동 제어 |

**통신:** ROS2 서비스 `Trigger` 1개(`/pick_place/run_once_package`) + `String` 상태 토픽 몇 개. (Action·커스텀 메시지는 사용하지 않음)

```
손님 → 키오스크 웹(:8000) → 키오스크 백엔드(큐/판정) ──Trigger──▶ 픽앤플레이스 노드 ─▶ 그리퍼
                                    │                                    ▲
                              JSON+SQLite 저장소          object_detector(YOLO+FastSAM) ─/detected_objects─┘
                                    │
                              락커 배정 + QR ─▶ 손님 수령
```

---

## 2. 주문 처리 흐름 (0 → 10)

라면 1개 주문 기준.

| 단계 | 담당 | 내용 |
|------|------|------|
| 0 | 대기 | 키오스크가 `/api/catalog`로 재고 표시, 로봇 `IDLE` |
| 1 | 손님→웹 | 라면 선택·주문 → `POST /api/orders` |
| 2 | 백엔드 | **서버에서 재고 재검증**(프론트 우회 차단) → 주문/아이템 생성, 상태 `QUEUED` |
| 3 | 백엔드 큐 루프 | `tick_queue()` 0.3초 주기 — 로봇 `IDLE` & 보류 아님이면 다음 아이템 선택 |
| 4 | 백엔드→로봇 | 클래스를 `/selected_object_label` 발행 + `/pick_place/run_once_package` 호출, 상태 `RUNNING` (5초 무응답 시 워치독 복구) |
| 5 | 검출 노드 | RealSense 영상 → YOLOv8(상품)+FastSAM(미학습) 인식, 다수결 안정화 |
| 6 | 픽 FSM | `DETECTING→PRE_PICK→PICK` — 연속 movel 하강 + 초음파 병렬 감시, 목표거리(기본 70mm) 도달 시 정지·집게 close |
| 7 | 픽 FSM | `LIFT` — 들어올리며 파지 검증. 실패 시 `HOME`으로 실패 처리 |
| 8 | 픽 FSM | `MOVE_TO_PLACE→PLACE→POST_PLACE` — 유저 주문은 sort 구역 무시, 전용 `package_position`에 적재 |
| 9 | 백엔드 | `/pick_place/cycle_result`(`success`/`dropped`/`failed`) + 상태 `IDLE` 복귀로 판정. `success`→아이템 `DONE`+통계 기록+**재고 −1** |
| 10 | 백엔드→손님 | 주문 완료 시 `assign_locker()`가 빈 락커+QR 토큰 배정(락커 `OCCUPIED`), WebSocket으로 수령 알림 → 키오스크가 QR 표시 → 손님 스캔(`/api/pickup/scan`→`confirm`) → 수령·종료 |

**입고 흐름:** 관리자가 `sort_all`로 물체 분류 시, 픽 노드가 분류한 클래스를 `/pick_place/sorted_class`로 발행 → 백엔드 **재고 +1**. 출고(−1)/입고(+1) 균형.

---

## 3. 기술 스택

- **로봇/센서:** 두산 E0509 + RH-P12-RN 그리퍼 + Intel RealSense RGB-D + 아두이노 초음파(HC-SR04)
- **비전:** YOLOv8 + FastSAM
- **파지:** 초음파 거리 기반 하강
- **통신:** ROS2 (Humble) — `Trigger` 서비스 + `String` 토픽
- **저장:** 하이브리드 — JSON(주문/큐/락커) + SQLite(재고/통계/이력, `~/.config/dsr_realsense_pick_place/store.db`)
- **웹:** React(Vite)+FastAPI(:8000), 관리자/로봇제어(:8080)

---

## 4. 실행 · 종료

```bash
# 전체 한 번에 (빌드+정리+로봇+키오스크+관리자)
bash mini_project/scripts/start_all.sh

# 픽앤플레이스만 (권장 래퍼 — 종료 시 shutdown 자동)
bash $(ros2 pkg prefix dsr_realsense_pick_place)/share/dsr_realsense_pick_place/scripts/run_pick_place_real.sh

# 직접 launch
ros2 launch dsr_realsense_pick_place pick_place.launch.py mode:=real

# 수동 종료
bash .../scripts/shutdown_nodes.sh --kill-launch
```

**웹 인터페이스**

| 페이지 | 주소 | 내용 |
|--------|------|------|
| 유저 키오스크 | `http://localhost:8000` | 주문(재고/품절 차단) · QR 수령(jsQR 폴백) |
| 관리자 패널 | `http://localhost:8000/admin` | 재고·락커·주문/큐·픽 통계·이력 |
| 로봇 제어 | `http://localhost:8080` | 그리퍼 전류/초음파·락커·로그 |

- **QR 수령:** 데스크톱 브라우저엔 `BarcodeDetector`가 없어 `jsQR` 폴백 사용. 카메라 접근(`getUserMedia`)은 `localhost`/HTTPS에서만 허용(원격 `http://IP` 차단).
- **재현성:** 모델(`models/*.pt`)·키오스크 빌드물(`web_kiosk/frontend/dist`)을 git에 포함 → clone 후 `npm run build` 없이 동작. 단 빌드는 두산 underlay(`ros2_ws`) 먼저 source 후 `--paths` 지정 필요.

---

## 5. 주요 패키지 · 스크립트

| 경로 | 역할 |
|------|------|
| `launch/pick_place.launch.py` | 전체 노드 런치 (`gui:=false`/`web:=true` 기본) |
| `web_kiosk/` | 유저 주문 키오스크 (React+FastAPI, :8000) |
| `dsr_realsense_pick_place/web_control_node.py` | 관리자 웹 제어 (SQLite+HTTP, :8080) |
| `dsr_realsense_pick_place/object_detector.py` | YOLO+FastSAM 검출 (클래스 다수결) |
| `dsr_realsense_pick_place/pick_place_node.py` | 픽 FSM (비동기 하강 / status3 방어) |
| `dsr_realsense_pick_place/task_repository.py` | 하이브리드 저장소(JSON+SQLite) |
| `dsr_gripper_tcp/` | 그리퍼 TCP 브릿지 |
| `scripts/run_pick_place_real.sh` | 권장 기동 래퍼 |
| `scripts/shutdown_nodes.sh` | 정상 종료 (DRCF/DRL 순서 해제) |
| `scripts/restart_gripper_bridge.sh` | 그리퍼만 복구 |
| `scripts/verify_pick_place_graph.sh` | 토픽·서비스·검출 연결 점검 |
| `config/pick_place_params.yaml` | 초음파·place·슬롯·zone 파라미터 |

---
