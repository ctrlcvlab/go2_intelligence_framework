# Implementation Plan: RViz2에서 Go2 Robot Model 표시

## Goal
현재 RViz2에서 `go2`의 TF 좌표계(`map`, `odom`, `base_link`, `camera_link`)만 보이고 실제 로봇 형상이 보이지 않는 문제를 해결한다.

최종 목표:
- RViz2에서 `RobotModel`로 Go2 메쉬/링크 형상 표시
- 현재 SLAM/Nav2 TF 트리와 충돌 없이 동작
- Isaac Sim 기준으로는 최소한 고정 자세 로봇 형상 표시
- 필요하면 추후 관절 상태(`joint_states`)까지 연동 가능한 구조 확보

---

## 현재 문제 정리

기존 문서와 설정을 기준으로 보면 현재 상태는 다음과 같다.

- `launch/go2_rtabmap.launch.py`에는 `static_transform_publisher`만 있고 `robot_state_publisher`가 없음
- `config/go2_sim.rviz`에는 `TF`, `Map`, `PointCloud2`, `LaserScan`은 있지만 `RobotModel` display가 없음
- 프로젝트 내부에는 Go2 URDF/mesh가 없고, 외부 워크스페이스 `/home/cvr/Desktop/sj/go2_ws/src/go2_description`를 참조해야 함

즉, 지금 RViz가 보여줄 수 있는 것은 TF 축뿐이다. RViz에서 로봇 형상이 보이려면 최소 3가지가 필요하다.

1. `robot_description` 파라미터
2. `robot_state_publisher`
3. RViz `RobotModel` display

추가로, 다리 관절까지 움직이는 모습을 보이려면 `joint_states`도 필요하다.

---

## 원인 분석

### 1. TF만 있고 Robot Model 입력이 없음
- TF display는 `/tf`, `/tf_static`만 있으면 축과 프레임 트리를 그릴 수 있다.
- 하지만 `RobotModel`은 URDF(`robot_description`)를 읽어 링크/메쉬를 렌더링해야 한다.
- 현재 launch에는 그 URDF를 공급하는 노드가 없다.

### 2. RViz 설정에 `RobotModel` display가 없음
- 현재 `config/go2_sim.rviz`에는 TF 표시만 활성화되어 있다.
- 따라서 launch에 `robot_state_publisher`를 추가하더라도 RViz 설정을 안 바꾸면 계속 안 보일 수 있다.

### 3. 고정 링크만 있는 현재 TF와 URDF 링크명이 일치해야 함
- 현재 TF 핵심 구조:
  - `map -> odom -> base_link -> camera_link -> camera_optical_frame`
- URDF도 `base_link`를 루트로 가져야 한다.
- `camera_link`를 URDF에 포함할지, 기존 static TF를 유지할지 역할을 정리해야 한다.

### 4. 관절 상태가 없으면 다리는 기본 자세로만 보임
- `robot_state_publisher`는 URDF와 `/joint_states`를 기반으로 각 링크 자세를 계산한다.
- `/joint_states`가 없으면 revolute joint는 기본값(보통 0 rad)로 렌더링된다.
- 즉, "로봇 형상 표시" 자체는 가능하지만 "실제 시뮬 자세 반영"은 별도 단계다.

---

## 목표 아키텍처

### 1단계 목표: 최소 구성으로 로봇 형상 보이기

```text
go2_description (URDF/Xacro)
        |
        v
robot_state_publisher
        |
        +--> /tf, /tf_static
        +--> robot_description

joint_state_publisher (또는 joint_state_publisher_gui 없이 기본값)
        |
        v
   /joint_states

RViz2
  - TF
  - RobotModel
  - Map / Scan / PointCloud
```

이 단계에서는 Go2 형상이 RViz에 보이는 것이 목적이다. 다리 관절은 기본 자세여도 괜찮다.

### 2단계 목표: Isaac Sim 관절 상태까지 반영

```text
Isaac Sim articulation state
        |
        v
custom joint_states publisher
        |
        v
   /joint_states
        |
        v
robot_state_publisher
        |
        v
RViz2 RobotModel
```

이 단계까지 가면 RViz에서 Go2 다리 움직임도 실제 시뮬과 맞출 수 있다.

---

## 구현 계획

## Phase 0: URDF 소스 검증

### 목표
RViz에서 바로 사용할 수 있는 Go2 URDF/Xacro와 mesh 경로가 정상인지 확인한다.

### 작업
- `/home/cvr/Desktop/sj/go2_ws/src/go2_description`를 RViz용 모델 소스로 사용
- 우선순위:
  1. `urdf/go2_description.urdf.xacro`
  2. `urdf/go2_description.urdf`
- 다음 항목 점검:
  - `base_link` 존재 여부
  - mesh 경로가 `package://go2_description/...` 형태로 정상인지
  - `ros2 pkg prefix go2_description`로 패키지 해석 가능한지
  - `check_urdf` 또는 `xacro` 변환이 통과하는지

### 확인 포인트
- 현재 `go2_description` 패키지에는 `dae/`와 `meshes/`가 모두 존재하므로, 실제 URDF가 참조하는 `dae/` 경로가 유효한지 확인 필요
- RViz는 package URI를 해석해야 하므로 해당 패키지가 ROS 환경에서 source 되어 있어야 함

### 완료 기준
- URDF/Xacro를 에러 없이 로드 가능
- 모든 mesh 파일이 깨지지 않고 resolve 가능

---

## Phase 1: `robot_state_publisher` 추가

### 목표
현재 SLAM launch에 Go2 모델 설명을 공급하는 노드를 추가한다.

### 구현 방향
- `launch/go2_rtabmap.launch.py`에 `robot_state_publisher` 노드 추가
- 필요하면 `launch/go2_navigation.launch.py`는 기존 `go2_rtabmap.launch.py` include를 통해 자동 상속
- `robot_description`는 `Command(["xacro", ...])` 또는 파일 read 방식으로 주입

### 권장 방식
`go2_description.urdf.xacro`를 우선 사용:

```python
robot_description = Command([
    "xacro ",
    "/home/cvr/Desktop/sj/go2_ws/src/go2_description/urdf/go2_description.urdf.xacro",
])
```

그리고:

```python
Node(
    package="robot_state_publisher",
    executable="robot_state_publisher",
    parameters=[{
        "robot_description": robot_description,
        "use_sim_time": use_sim_time,
    }],
)
```

### 주의사항
- 이미 `base_link -> camera_link -> camera_optical_frame` static TF가 있으므로, URDF 안에 동일한 camera chain을 넣을지 말지 먼저 결정해야 함
- 현재는 중복 TF 충돌을 피하기 위해 다음 방식을 권장:
  - Go2 본체/다리 링크는 URDF에서 발행
  - 카메라 프레임은 기존 static TF 유지

### 완료 기준
- `robot_state_publisher` 실행 후 `/robot_description` 및 추가 링크 TF가 생김
- 기존 `map -> odom -> base_link` 체인과 충돌 없음

---

## Phase 2: `joint_states` 최소 구성 추가

### 목표
RViz `RobotModel`이 Go2의 다리 링크들을 렌더링할 수 있도록 최소 joint state를 공급한다.

### 선택지

1. `joint_state_publisher` 사용
- 가장 빠른 방법
- 실제 시뮬 자세는 반영되지 않음
- RViz에서 기본 자세 확인용으로 적합

2. Isaac Sim에서 실제 관절 상태를 `/joint_states`로 발행
- 구현량 큼
- RViz와 시뮬 자세가 일치
- 추후 확장 단계로 적합

### 권장 순서
- 1차: `joint_state_publisher`로 먼저 형상 표시 성공
- 2차: 필요 시 `scripts/go2_sim.py` 또는 별도 ROS 브릿지 노드에서 실제 관절 상태 발행

### 완료 기준
- RViz에서 몸통만이 아니라 다리 링크까지 포함한 전체 Go2 형상이 표시됨

---

## Phase 3: RViz 설정 보강

### 목표
현재 RViz 설정 파일에 `RobotModel` display를 추가한다.

### 작업
- `config/go2_sim.rviz`에 `rviz_default_plugins/RobotModel` 추가
- Description Source는 `robot_description`
- Visual Enabled / Collision Enabled 적절히 선택
- Fixed Frame은 기존처럼 `map` 유지

### 권장 표시 구성
- `TF`: ON
- `RobotModel`: ON
- `Map`: ON
- `LaserScan`: ON
- `PointCloud2`: ON

### 완료 기준
- RViz 실행 직후 `RobotModel` 항목이 보이고 상태가 `OK`
- TF 축 위에 실제 Go2 메쉬 또는 링크 형상이 겹쳐서 표시됨

---

## Phase 4: TF 충돌 및 프레임 정합 검증

### 목표
URDF 기반 링크와 기존 static TF/RTAB-Map TF가 충돌하지 않도록 정리한다.

### 검증 항목
- `base_link`가 중복 발행되지 않는지 확인
- `camera_link`, `camera_optical_frame`이 URDF와 static TF에서 동시에 나오지 않는지 확인
- `map -> odom`은 RTAB-Map
- `odom -> base_link`는 Isaac Sim odometry
- `base_link` 이하 본체 링크들은 `robot_state_publisher`

### 권장 원칙
- 위치 추정용 프레임과 시각화용 링크 프레임의 책임을 분리
- 루트 이동 프레임은 기존 시스템 유지:
  - `map -> odom -> base_link`
- 본체 형상 표현은 `base_link` 하위 링크만 URDF가 담당

### 완료 기준
- `view_frames` 기준으로 중복/루프 없는 트리 확인
- RViz에서 robot model이 튀거나 사라지지 않음

---

## Phase 5: Isaac Sim 실제 관절 자세 연동 (선택)

### 목표
RViz 로봇 모델이 실제 Go2 시뮬 자세와 동일하게 움직이도록 만든다.

### 구현 후보
- `scripts/go2_sim.py`에서 articulation joint positions 읽기
- ROS2 `/joint_states` 메시지로 퍼블리시
- joint name은 URDF와 정확히 일치해야 함

### 주의사항
- 현재 핵심 문제는 "형상이 안 보임"이므로 이 단계는 필수 아님
- 우선 형상 표시부터 해결한 뒤 진행하는 것이 맞다

### 완료 기준
- 보행 중 RViz의 다리 움직임이 Isaac Sim과 대체로 일치

---

## 권장 작업 순서

1. `go2_description` URDF/Xacro 단독 로드 검증
2. `go2_rtabmap.launch.py`에 `robot_state_publisher` 추가
3. `joint_state_publisher` 추가
4. `config/go2_sim.rviz`에 `RobotModel` display 추가
5. RViz에서 형상 표시 확인
6. TF 충돌 없으면 Nav2 launch까지 동일 구조 확장
7. 필요 시 실제 `/joint_states` 연동

---

## 예상 이슈와 대응

### 이슈 1: 메쉬는 안 보이고 링크만 보임
- 원인:
  - mesh 파일 경로 해석 실패
  - `go2_description` 패키지가 source 안 됨
- 대응:
  - `source /home/cvr/Desktop/sj/go2_ws/install/setup.bash`
  - `package://go2_description/dae/...` 실제 존재 여부 확인

### 이슈 2: RobotModel 상태가 Error
- 원인:
  - `robot_description` 미주입
  - URDF 파싱 실패
- 대응:
  - `ros2 param get /robot_state_publisher robot_description`
  - `check_urdf` / `xacro` 검증

### 이슈 3: 카메라 TF가 중복됨
- 원인:
  - URDF 안의 camera 링크와 static TF 중복
- 대응:
  - 초기에는 카메라 링크를 기존 static TF로만 유지

### 이슈 4: 다리 자세가 부자연스럽거나 안 움직임
- 원인:
  - `/joint_states` 부재
- 대응:
  - 1차는 허용
  - 2차로 Isaac Sim joint state publisher 추가

---

## 체크리스트

- [ ] `go2_description` 패키지 ROS 환경에서 인식 확인
- [ ] URDF/Xacro 파싱 확인
- [ ] `robot_state_publisher` launch 추가
- [ ] `joint_state_publisher` launch 추가
- [ ] `config/go2_sim.rviz`에 `RobotModel` 추가
- [ ] RViz에서 Go2 본체 형상 표시 확인
- [ ] TF 중복 여부 확인
- [ ] Nav2 launch에서도 동일하게 표시 확인

---

## 결론

지금 RViz에 TF만 보이고 실제 Go2가 안 보이는 이유는 단순히 "시각화할 로봇 모델 입력"이 빠져 있기 때문이다.

핵심 해결책은 아래 3가지다.

1. `go2_description` URDF/Xacro를 `robot_description`으로 공급
2. `robot_state_publisher`와 최소 `joint_states` 추가
3. `config/go2_sim.rviz`에 `RobotModel` display 추가

우선은 "고정 자세라도 Go2 형상을 보이게 하는 것"을 1차 목표로 두고, 그 다음 단계에서 Isaac Sim 실제 관절 자세 연동으로 확장하는 순서가 가장 안전하다.
