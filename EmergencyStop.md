##  목표

정적 장애물 회피와 별도로, 로봇 전방 가까운 거리에서 갑자기 장애물이 나타났을 때 로봇이 즉시 정지하고, 장애물이 제거되면 일정 시간 대기 후 주행을 재개하는 기능을 구현한다.

이 기능은 “동적 장애물 회피”보다는 **갑툭튀 장애물 Emergency Stop**으로 정의한다.

---

##  역할 구분

| 기능 | 담당 |
| --- | --- |
| 정적 장애물 우회 | Nav2 local costmap |
| 경로 계획 | planner_server |
| 경로 추종 | controller_server |
| 갑툭튀 장애물 정지 | setros_safety_node |
| 최종 속도 명령 | `/cmd_vel` |

---

##  최종 속도 명령 구조

기존 구조:

```
Nav2 controller_server
    ↓
/cmd_vel_nav
    ↓
topic_tools relay
    ↓
/cmd_vel
    ↓
turtlebot3_node
```

최종 구조:

```
Nav2 controller_server
    ↓
/cmd_vel_nav
    ↓
setros_safety_node
    ↓
/cmd_vel
    ↓
turtlebot3_node
```

`safety_node`가 `/cmd_vel_nav`를 받아 `/cmd_vel`로 전달하되, 전방 가까운 장애물이 감지되면 속도 명령을 0으로 차단한다.

---

##  Safety Node 입력/출력

| 구분 | 토픽 | 메시지 타입 | 역할 |
| --- | --- | --- | --- |
| 입력 | `/cmd_vel_nav` | `geometry_msgs/msg/TwistStamped` | Nav2 주행 명령 |
| 입력 | `/scan` | `sensor_msgs/msg/LaserScan` | 전방 장애물 거리 판단 |
| 출력 | `/cmd_vel` | `geometry_msgs/msg/TwistStamped` | 최종 속도 명령 |
| 출력 | `/safety_state` | `std_msgs/msg/String` | safety 상태 표시 |

---

##  Emergency Stop 파라미터

최종 시연용 설정값:

```
front_angle_deg: 25.0
stop_distance: 0.28
resume_delay: 1.0
```

의미:

| 파라미터 | 값 | 의미 |
| --- | --- | --- |
| `front_angle_deg` | 25.0 | 로봇 정면 ±25도 범위 감시 |
| `stop_distance` | 0.28 | 전방 28cm 이내 장애물 감지 시 정지 |
| `resume_delay` | 1.0 | 장애물 제거 후 1초 대기 후 재출발 |

이 값은 정적 장애물 우회 상황에서 safety node가 과도하게 개입하지 않도록 줄인 값이다.

---

##  Safety State 정의

| 상태 | 의미 |
| --- | --- |
| `NORMAL` | 정상 주행 |
| `SUDDEN_OBSTACLE_STOP` | 갑툭튀 장애물 감지, 즉시 정지 |
| `WAIT_AFTER_SUDDEN_STOP` | 장애물 제거 후 재출발 전 대기 |

상태 확인:

```
ros2 topicecho /safety_state
```

---

##  Emergency Stop 테스트 방법

1. Nav2 Goal을 찍어 로봇을 주행시킨다.
2. 로봇 전방 가까운 위치에 박스를 갑자기 넣는다.
3. `/safety_state`가 `SUDDEN_OBSTACLE_STOP`으로 바뀌는지 확인한다.
4. 로봇이 실제로 정지하는지 확인한다.
5. 박스를 제거한다.
6. `/safety_state`가 `WAIT_AFTER_SUDDEN_STOP`으로 바뀌는지 확인한다.
7. 1초 후 `NORMAL`로 돌아오고 주행이 재개되는지 확인한다.

기대 상태 변화:

```
NORMAL
→ SUDDEN_OBSTACLE_STOP
→ WAIT_AFTER_SUDDEN_STOP
→ NORMAL
```

---

##  속도 차단 확인

Emergency Stop 상태에서 `/cmd_vel`을 확인한다.

```
ros2 topicecho /cmd_vel
```

정상이라면 정지 상태에서 다음과 같이 출력된다.

```
twist:
  linear:
    x: 0.0
    y: 0.0
    z: 0.0
  angular:
    x: 0.0
    y: 0.0
    z: 0.0
```

즉 Nav2가 `/cmd_vel_nav`로 주행 명령을 생성하더라도, safety node가 최종 `/cmd_vel`을 0으로 차단한다.

---

##  Emergency Stop 결과

테스트 결과, 로봇은 주행 중 전방 갑툭튀 장애물에 대해 정상적으로 정지하였다.

정리:

```
Emergency Stop 성공
- /scan 기반 전방 거리 감지 정상
- /safety_state 상태 변화 정상
- /cmd_vel 차단 정상
- 장애물 제거 후 대기
- 자동 주행 재개 정상
```
