urtleBot3 Burger 실물 SLAM 및 Nav2 자율주행 구현 기록
<img width="1586" height="1511" alt="KakaoTalk_20260527_121659041 jpg" src="https://github.com/user-attachments/assets/dfeb323e-6089-452e-814d-7f0e120165f4" />

SLAM 이후 Localization
## 1. 프로젝트 개요

### 프로젝트명

**실물 TurtleBot3 Burger 기반 SLAM 및 장애물 회피 자율주행 시스템 구현**

### 목표

실물 TurtleBot3 Burger를 이용해 다음 흐름을 구현한다.

```
Bringup
→ LiDAR 확인
→ Teleop 수동 주행
→ Cartographer SLAM
→ 지도 저장
→ 저장 지도 기반 Localization
→ Nav2 2D Goal Pose 자율주행
→ local costmap 기반 장애물 회피
```

### 사용 환경

| 구분 | 환경 |
| --- | --- |
| 로봇 | TurtleBot3 Burger |
| Raspberry Pi | Ubuntu 24.04, ROS 2 Jazzy |
| Remote PC | Ubuntu 24.04, ROS 2 Jazzy |
| ROS_DOMAIN_ID | 30 |
| TURTLEBOT3_MODEL | burger |
| LDS_MODEL | LDS-01 |
| SLAM | turtlebot3_cartographer |
| Navigation | turtlebot3_navigation2, Nav2 |

환경변수:

```
exportROS_DOMAIN_ID=30
exportTURTLEBOT3_MODEL=burger
exportLDS_MODEL=LDS-01
```

---

# 2. 전체 성공 흐름

## 2.1 TurtleBot3 Bringup

TurtleBot Raspberry Pi에서 bringup 실행:

```
ros2 launch turtlebot3_bringup robot.launch.py
```

정상 확인 토픽:

```
/scan
/odom
/tf
/tf_static
/battery_state
/imu
/joint_states
/cmd_vel
```

확인 명령:

```
ros2 topic list
```

LiDAR 확인:

```
ros2 topicecho /scan--once
```

정상 기준:

```
frame_id: base_scan
ranges: [...]
```

---

## 2.2 Teleop 수동 주행 확인

Remote PC에서 실행:

```
ros2 run turtlebot3_teleop teleop_keyboard
```

확인 결과:

```
키보드 입력으로 바퀴 정상 동작
전진/후진/회전 가능
OpenCR, 모터, 배터리, bringup 정상 판단
```

이 단계에서 teleop이 정상 동작했기 때문에, 이후 Nav2 주행 실패가 발생했을 때 **모터나 OpenCR 자체 문제는 낮은 가능성**으로 판단할 수 있었다.

---

## 2.3 Cartographer SLAM 실행

Remote PC에서 SLAM 실행:

```
ros2 launch turtlebot3_cartographer cartographer.launch.py
```

SLAM 중에는 teleop으로 로봇을 천천히 움직이며 지도를 생성했다.

```
ros2 run turtlebot3_teleop teleop_keyboard
```

정상 상태:

```
RViz 실행
LaserScan / PointCloud 표시
로봇 이동에 따라 지도 생성
벽면이 점진적으로 map에 반영됨
```

---

## 2.4 지도 저장

SLAM 완료 후 지도 저장:

```
ros2 run nav2_map_server map_saver_cli-f ~/setros_map_home
```

저장 확인:

```
ls ~ |grep setros_map_home
```

정상 생성 파일:

```
setros_map_home.pgm
setros_map_home.yaml
```

---

# 3. Nav2 실행 및 Localization

## 3.1 Nav2 실행

SLAM과 teleop은 종료하고, TurtleBot bringup은 유지한 상태에서 Remote PC에서 Nav2 실행:

```
ros2 launch turtlebot3_navigation2 navigation2.launch.py map:=$HOME/setros_map_home.yaml
```

---

## 3.2 2D Pose Estimate 설정

RViz에서 **2D Pose Estimate**를 사용해 실제 로봇 위치와 방향을 지도 위에 맞췄다.

정상 판단 기준:

```
보라색 AMCL particle이 로봇 주변 또는 벽면에 잘 분포
LaserScan / PointCloud가 지도 벽면과 겹침
로봇의 방향이 실제 방향과 유사함
```

TF 확인:

```
ros2 topic list |grep tf
ros2 run tf2_ros tf2_echo map base_footprint
```

정상 판단:

```
/tf, /tf_static 존재
map → base_footprint 변환 출력
```

---

# 4. 의미 있었던 문제 상황 및 해결 과정

## 문제 1. Nav2 Goal을 찍었는데 `/navigate_to_pose` action server가 없음

### 증상

RViz에서 Nav2 Goal을 찍으면 다음 에러 발생:

```
navigate_to_pose action server is not available. Is the initial pose set?
```

확인 명령:

```
ros2 action info /navigate_to_pose
```

결과:

```
Action servers: 0
```

### 원인 분석

`/navigate_to_pose` action server는 `bt_navigator`가 active 상태일 때 제공된다.

따라서 lifecycle 상태를 확인했다.

```
ros2 lifecycleget /bt_navigator
```

결과:

```
unconfigured
```

즉, Nav2 노드들이 실행은 되었지만 **lifecycle configure/activate가 되지 않은 상태**였다.

### 해결

Nav2 lifecycle 노드들을 수동으로 configure → activate 했다.

```
ros2 lifecycleset /map_server configure
ros2 lifecycleset /map_server activate

ros2 lifecycleset /amcl configure
ros2 lifecycleset /amcl activate

ros2 lifecycleset /planner_server configure
ros2 lifecycleset /planner_server activate

ros2 lifecycleset /controller_server configure
ros2 lifecycleset /controller_server activate

ros2 lifecycleset /behavior_server configure
ros2 lifecycleset /behavior_server activate

ros2 lifecycleset /bt_navigator configure
ros2 lifecycleset /bt_navigator activate

ros2 lifecycleset /velocity_smoother configure
ros2 lifecycleset /velocity_smoother activate
```

확인:

```
ros2 lifecycleget /bt_navigator
ros2 action info /navigate_to_pose
```

정상 결과:

```
bt_navigator: active
Action servers: 1
```

### 정리

이 문제의 핵심은 RViz나 Goal 설정 문제가 아니라, **Nav2 lifecycle 노드가 active 상태가 아니었던 것**이다.

향후에는 수동 activate 대신 Nav2 실행 시 다음 옵션을 사용할 계획이다.

```
autostart:=true
```

예상 실행 명령:

```
ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  map:=$HOME/setros_map_home.yaml \
  params_file:=$HOME/setros_nav2_params.yaml \
  autostart:=true
```

---

## 문제 2. Nav2 Goal은 active가 되지만 로봇이 움직이지 않음

### 증상

Nav2 lifecycle 활성화 후 `/navigate_to_pose` action server는 정상적으로 살아났다.

하지만 RViz에서 Nav2 Goal을 찍어도 로봇이 움직이지 않았다.

확인:

```
ros2 topicecho /cmd_vel_nav
ros2 topicecho /cmd_vel
```

결과:

```
/cmd_vel_nav에는 0이 아닌 속도 명령이 출력됨
/cmd_vel은 무한대기
로봇은 움직이지 않음
```

### 원인 분석

이 시점에서 다음은 정상으로 판단했다.

```
SLAM 정상
AMCL localization 정상
planner 정상
controller_server 정상
Nav2 Goal 처리 정상
```

왜냐하면 `/cmd_vel_nav`에 실제 속도 명령이 나오고 있었기 때문이다.

문제는 `/cmd_vel_nav`에서 생성된 속도 명령이 최종 `/cmd_vel`로 전달되지 않는 것이었다.

---

## 문제 3. `/cmd_vel` publisher가 없음

### 확인 명령

```
ros2 topic info /cmd_vel-v
```

결과:

```
Type: geometry_msgs/msg/TwistStamped

Publisher count: 0
Subscription count: 1

Node name: turtlebot3_node
Topic type: geometry_msgs/msg/TwistStamped
Endpoint type: SUBSCRIPTION
```

### 해석

```
turtlebot3_node는 /cmd_vel을 구독하고 있음
하지만 /cmd_vel을 발행하는 노드가 없음
따라서 실제 로봇으로 속도 명령이 전달되지 않음
```

즉, TurtleBot3 노드는 `/cmd_vel`을 기다리고 있었지만 Nav2는 최종적으로 `/cmd_vel`에 아무것도 발행하지 않고 있었다.

---

## 문제 4. `/cmd_vel_nav`는 정상 출력됨

### 확인 명령

```
ros2 topic info /cmd_vel_nav-v
```

결과 요약:

```
Type: geometry_msgs/msg/TwistStamped
Publisher count: 6
Subscription count: 1

Publisher:
- controller_server
- behavior_server

Subscriber:
- velocity_smoother
```

### 해석

```
controller_server는 /cmd_vel_nav에 속도 명령을 정상 발행
velocity_smoother는 /cmd_vel_nav를 구독 중
하지만 최종 /cmd_vel로 출력되지 않음
```

따라서 문제는 다음 중 하나로 좁혀졌다.

```
velocity_smoother output remap 문제
Nav2 launch remap 문제
params YAML의 cmd_vel 관련 설정 문제
/cmd_vel_nav → /cmd_vel 연결 누락
```

---

# 5. 임시 해결: topic_tools relay

## 해결 명령

Remote PC 새 터미널에서 실행:

```
ros2 run topic_tools relay /cmd_vel_nav /cmd_vel
```

## 결과

relay 실행 후 RViz에서 Nav2 Goal을 찍자 로봇이 실제로 움직였다.

성공 구조:

```
/controller_server
  ↓
/cmd_vel_nav
  ↓
topic_tools relay
  ↓
/cmd_vel
  ↓
/turtlebot3_node
  ↓
실제 TurtleBot3 주행
```

## 의미

이 결과로 다음이 확정되었다.

```
Nav2 planner/controller는 정상
AMCL localization 정상
TurtleBot3 motor/OpenCR 정상
문제는 /cmd_vel_nav → /cmd_vel 연결 누락
```

현재 relay는 임시 해결책이지만, 실제 시연 기준으로는 안정적으로 동작했다.

---

# 6. 주행 품질 튜닝

## 6.1 초기 주행 특성

relay 적용 후 로봇은 목적지까지 이동했지만, 다음 특성이 있었다.

```
살짝 곡선으로 이동
목적지 도착 성공
출발 시 급가속 느낌 있음
곡선은 완만하게 형성됨
```

이 정도는 Nav2 주행에서 정상 범위로 판단했다.

다만 시연 안정성을 위해 속도와 가속도를 낮췄다.

---

## 6.2 DWB Controller 파라미터 확인

확인 명령:

```
ros2 param list /controller_server |grep-E"FollowPath.*(vel|acc|decel|speed|theta)"
```

확인된 주요 파라미터:

```
FollowPath.acc_lim_theta
FollowPath.acc_lim_x
FollowPath.decel_lim_theta
FollowPath.decel_lim_x
FollowPath.max_speed_xy
FollowPath.max_vel_theta
FollowPath.max_vel_x
FollowPath.min_speed_theta
FollowPath.min_speed_xy
FollowPath.min_vel_x
FollowPath.trans_stopped_velocity
FollowPath.vtheta_samples
```

이를 통해 현재 controller가 DWB 계열 파라미터를 사용한다고 판단했다.

---

## 6.3 적용한 시연용 속도 튜닝값

```
ros2 paramset /controller_server FollowPath.max_vel_x0.10
ros2 paramset /controller_server FollowPath.max_speed_xy0.10
ros2 paramset /controller_server FollowPath.acc_lim_x0.15
ros2 paramset /controller_server FollowPath.max_vel_theta0.6
ros2 paramset /controller_server FollowPath.acc_lim_theta0.8
```

각 파라미터 의미:

| 파라미터 | 의미 |
| --- | --- |
| `max_vel_x` | 직진 최대 속도 |
| `max_speed_xy` | 평면 이동 최대 속도 |
| `acc_lim_x` | 직진 가속 제한 |
| `max_vel_theta` | 회전 최대 속도 |
| `acc_lim_theta` | 회전 가속 제한 |

적용 결과:

```
출발이 더 안정적
급가속 감소
목표 지점까지 정상 도착
시연 가능한 주행 품질 확보
```

---

# 7. Costmap Inflation 튜닝

## 문제 인식

실험 공간이 좁은 하드보드/가벽 기반 코스였기 때문에, 기본 inflation radius가 너무 크면 Nav2가 통로를 지나갈 수 없는 공간으로 판단할 가능성이 있었다.

초기에는 `inflation_radius`를 낮춰 주행 가능성을 확보했다.

최종적으로는 너무 벽에 붙지 않도록 `0.15`를 시연용 값으로 사용했다.

## 적용 명령

```
ros2 paramset /local_costmap/local_costmap inflation_layer.inflation_radius0.15
ros2 paramset /global_costmap/global_costmap inflation_layer.inflation_radius0.15
```

현재 권장값:

```
robot_radius: 0.1
footprint: []
inflation_radius: 0.15
cost_scaling_factor: 5.0
```

---

# 8. 현재 최종 실행 구조

## TurtleBot Pi

```
ros2 launch turtlebot3_bringup robot.launch.py
```

## Remote PC - Nav2

```
ros2 launch turtlebot3_navigation2 navigation2.launch.py map:=$HOME/setros_map_home.yaml
```

## Remote PC - relay

```
ros2 run topic_tools relay /cmd_vel_nav /cmd_vel
```

## RViz

```
1. 2D Pose Estimate로 현재 위치 설정
2. AMCL particle과 LaserScan 정합 확인
3. Nav2 Goal 지정
4. 로봇 주행 확인
```

---

# 9. 현재 성공 상태 요약

| 항목 | 상태 |
| --- | --- |
| TurtleBot3 bringup | 성공 |
| LiDAR `/scan` | 정상 |
| Teleop 수동 주행 | 성공 |
| Cartographer SLAM | 성공 |
| 지도 저장 | 성공 |
| 저장 지도 기반 Nav2 실행 | 성공 |
| AMCL localization | 성공 |
| `/navigate_to_pose` action server | lifecycle activate 후 성공 |
| `/cmd_vel_nav` 생성 | 성공 |
| `/cmd_vel` 전달 | relay로 해결 |
| RViz Nav2 Goal 주행 | 성공 |
| 속도 튜닝 | 성공 |
| 시연 가능성 | 가능 |

---

# 10. 남은 개선 과제

## 10.1 Lifecycle 자동 활성화

현재는 수동으로 configure/activate를 수행했다.

향후에는 Nav2 실행 시 `autostart:=true`를 적용해 자동 활성화할 계획이다.

예상 명령:

```
ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  map:=$HOME/setros_map_home.yaml \
  params_file:=$HOME/setros_nav2_params.yaml \
  autostart:=true
```

확인 명령:

```
ros2 lifecycleget /bt_navigator
ros2 action info /navigate_to_pose
```

정상 기준:

```
bt_navigator: active
Action servers: 1
```

---

## 10.2 튜닝 파라미터 YAML 영구 저장

현재 런타임에서 `ros2 param set`으로 적용한 값은 재실행 시 사라질 수 있다.

따라서 별도의 Nav2 params YAML 파일에 저장해야 한다.

기본 파일 복사:

```
cp /opt/ros/jazzy/share/turtlebot3_navigation2/param/burger.yaml ~/setros_nav2_params.yaml
```

수정할 주요 값:

```
controller_server:
  ros__parameters:
    FollowPath:
      max_vel_x: 0.10
      max_speed_xy: 0.10
      acc_lim_x: 0.15
      max_vel_theta: 0.6
      acc_lim_theta: 0.8

local_costmap:
  local_costmap:
    ros__parameters:
      inflation_layer:
        inflation_radius: 0.15

global_costmap:
  global_costmap:
    ros__parameters:
      inflation_layer:
        inflation_radius: 0.15
```

반영 확인:

```
ros2 paramget /controller_server FollowPath.max_vel_x
ros2 paramget /controller_server FollowPath.max_speed_xy
ros2 paramget /controller_server FollowPath.acc_lim_x
ros2 paramget /controller_server FollowPath.max_vel_theta
ros2 paramget /controller_server FollowPath.acc_lim_theta
ros2 paramget /local_costmap/local_costmap inflation_layer.inflation_radius
ros2 paramget /global_costmap/global_costmap inflation_layer.inflation_radius
```

---

## 10.3 `/cmd_vel_nav → /cmd_vel` 연결 영구화

현재는 relay를 수동 실행하고 있다.

```
ros2 run topic_tools relay /cmd_vel_nav /cmd_vel
```

향후 개선 방향은 2가지다.

### 방법 A. 시연 안정성 우선

relay를 launch 파일에 포함해 자동 실행한다.

구조:

```
Nav2 실행
+ topic_tools relay /cmd_vel_nav /cmd_vel 자동 실행
```

장점:

```
이미 실제 주행 성공으로 검증됨
시연 안정성이 높음
수정 범위가 작음
```

단점:

```
velocity_smoother를 정석적으로 활용하지 못할 수 있음
구조적으로 임시 해결에 가까움
```

### 방법 B. 정석 수정

`velocity_smoother`의 출력이 최종 `/cmd_vel`로 가도록 Nav2 launch 또는 params remap을 수정한다.

목표 구조:

```
/controller_server
  ↓
/cmd_vel_nav
  ↓
/velocity_smoother
  ↓
/cmd_vel
  ↓
/turtlebot3_node
```

장점:

```
Nav2 구조에 더 적합
속도 smoothing을 정상적으로 활용 가능
급가속 완화에 유리
```

단점:

```
launch/remap/YAML 구조를 정확히 확인해야 함
디버깅 시간이 더 필요함
```

---

# 11. 이번 작업에서 얻은 핵심 디버깅 포인트

## 11.1 Teleop 성공 여부는 매우 중요함

teleop이 정상 동작하면 다음 항목들은 정상으로 볼 수 있다.

```
배터리
OpenCR
모터
TurtleBot bringup
/cmd_vel 구독 경로
기본 주행 하드웨어
```

따라서 이후 Nav2 주행 실패는 하드웨어보다 Nav2 설정, lifecycle, topic remap 문제로 좁힐 수 있었다.

---

## 11.2 Nav2 Goal 실패 시 action server부터 확인해야 함

RViz에서 Goal이 안 먹히면 먼저 확인할 것:

```
ros2 action info /navigate_to_pose
```

`Action servers: 0`이면 RViz 문제가 아니라 Nav2 lifecycle 문제일 가능성이 높다.

---

## 11.3 Nav2가 움직이지 않을 때 `/cmd_vel_nav`와 `/cmd_vel`을 분리해서 봐야 함

단순히 `/cmd_vel`만 보면 controller가 문제인지, 전달 체인이 문제인지 구분하기 어렵다.

이번에는 다음처럼 분리해서 원인을 찾았다.

```
ros2 topicecho /cmd_vel_nav
ros2 topicecho /cmd_vel
```

판단:

```
/cmd_vel_nav 값 있음 + /cmd_vel 없음
→ Nav2 controller는 정상
→ 최종 cmd_vel 전달 체인 문제
```

---

## 11.4 `ros2 topic info -v`가 원인 확정에 중요했음

```
ros2 topic info /cmd_vel-v
```

이 명령으로 다음을 확인했다.

```
/cmd_vel 타입
Publisher count
Subscription count
구독 노드 이름
```

이번 케이스에서는:

```
Publisher count: 0
Subscription count: 1
Subscriber: turtlebot3_node
```

이었기 때문에, 로봇이 안 움직인 이유가 명확해졌다.

---

# 12. 최종 결론

이번 작업을 통해 실물 TurtleBot3 Burger에서 다음 전체 흐름을 성공했다.

```
Bringup
→ Teleop
→ SLAM
→ Map Save
→ Localization
→ Nav2 Goal
→ /cmd_vel relay
→ 실제 자율주행 성공
```

핵심 문제는 Nav2 자체가 아니라, **Nav2 controller가 생성한 `/cmd_vel_nav`가 최종 `/cmd_vel`로 전달되지 않는 topic 연결 문제**였다.

임시 해결로 `topic_tools relay`를 사용해 실제 주행에 성공했고, 이후 속도/가속도 및 costmap inflation 튜닝을 통해 시연 가능한 주행 품질을 확보했다.

향후 작업은 다음 3가지다.

```
1. autostart:=true로 Nav2 lifecycle 자동 활성화
2. setros_nav2_params.yaml에 튜닝 파라미터 영구 저장
3. relay 자동 실행 또는 velocity_smoother remap 정석 수정
```

이번 기록은 기존 문제 정리에서 확인된 `/cmd_vel_nav`는 생성되지만 최종 `/cmd_vel`이 비어 로봇이 움직이지 않던 문제를 실제 실험에서 relay로 해결한 과정이다.
