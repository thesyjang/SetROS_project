## 1. 패키지 구조

```
~/setros_ws/src/setros_safety
├── launch
│   └── setros_nav2_safety.launch.py
├── package.xml
├── setup.py
└── setros_safety
    ├── __init__.py
    └── safety_cmd_vel_node.py
```

---

## 2. Safety Node

경로:

```
~/setros_ws/src/setros_safety/setros_safety/safety_cmd_vel_node.py
```

역할:

```
Nav2의 /cmd_vel_nav를 받아 최종 /cmd_vel로 전달한다.
단, LiDAR /scan 기준으로 전방 갑툭튀 장애물이 감지되면 /cmd_vel을 0으로 차단한다.
```

최종 코드:

```python
import math
import time

import rclpy
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data

from geometry_msgs.msg import TwistStamped
from sensor_msgs.msg import LaserScan
from std_msgs.msg import String

class SafetyCmdVelNode(Node):
    def __init__(self):
        super().__init__('setros_safety_node')

        # ===== Safety parameters =====
        # front_angle_deg: 로봇 정면 기준 감시 각도 범위
        # stop_distance: 이 거리 안에 장애물이 들어오면 정지
        # resume_delay: 장애물 제거 후 재출발 전 대기 시간
        self.declare_parameter('front_angle_deg', 25.0)
        self.declare_parameter('stop_distance', 0.28)
        self.declare_parameter('resume_delay', 1.0)

        self.front_angle_deg = float(self.get_parameter('front_angle_deg').value)
        self.stop_distance = float(self.get_parameter('stop_distance').value)
        self.resume_delay = float(self.get_parameter('resume_delay').value)

        # ===== Internal state =====
        self.latest_cmd = TwistStamped()
        self.obstacle_detected = False
        self.last_obstacle_time = 0.0
        self.state = 'NORMAL'

        # 디버깅용: 마지막 전방 최소 거리/각도 저장
        self.last_min_front_distance = float('inf')
        self.last_min_front_angle = 0.0

        # ===== Subscribers =====
        # Nav2 controller_server가 만든 원본 속도 명령
        self.cmd_sub = self.create_subscription(
            TwistStamped,
            '/cmd_vel_nav',
            self.cmd_callback,
            10
        )

        # LiDAR scan 구독
        # TurtleBot3 /scan은 sensor-data QoS를 사용하므로 qos_profile_sensor_data 필요
        self.scan_sub = self.create_subscription(
            LaserScan,
            '/scan',
            self.scan_callback,
            qos_profile_sensor_data
        )

        # ===== Publishers =====
        # TurtleBot3가 실제로 구독하는 최종 속도 명령
        self.cmd_pub = self.create_publisher(
            TwistStamped,
            '/cmd_vel',
            10
        )

        # safety 상태 확인용 토픽
        self.state_pub = self.create_publisher(
            String,
            '/safety_state',
            10
        )

        # 20Hz로 최종 /cmd_vel 출력
        self.timer = self.create_timer(0.05, self.timer_callback)

        self.get_logger().info('setros_safety_node started')
        self.get_logger().info(f'front_angle_deg={self.front_angle_deg}')
        self.get_logger().info(f'stop_distance={self.stop_distance}')
        self.get_logger().info(f'resume_delay={self.resume_delay}')

    def cmd_callback(self, msg):
        # Nav2가 만든 최신 속도 명령 저장
        self.latest_cmd = msg

    def normalize_angle_deg(self, angle_deg):
        # LiDAR 각도를 -180 ~ +180도 범위로 정규화
        # /scan이 0~360도 형식으로 들어와도 전방 판단 가능
        return ((angle_deg + 180.0) % 360.0) - 180.0

    def scan_callback(self, msg):
        front_ranges = []

        for i, r in enumerate(msg.ranges):
            # 유효하지 않은 거리값 제거
            if math.isinf(r) or math.isnan(r):
                continue

            if r < msg.range_min or r > msg.range_max:
                continue

            # 현재 scan index의 각도 계산
            angle_rad = msg.angle_min + i * msg.angle_increment
            angle_deg = math.degrees(angle_rad)
            angle_deg = self.normalize_angle_deg(angle_deg)

            # 정면 ±front_angle_deg 범위만 감시
            if abs(angle_deg) <= self.front_angle_deg:
                front_ranges.append((angle_deg, r))

        if not front_ranges:
            self.obstacle_detected = False
            self.last_min_front_distance = float('inf')
            self.last_min_front_angle = 0.0
            return

        # 전방 영역에서 가장 가까운 장애물 추출
        min_angle, min_distance = min(front_ranges, key=lambda x: x[1])

        self.last_min_front_distance = min_distance
        self.last_min_front_angle = min_angle

        # 정지 거리 안에 들어오면 갑툭튀 장애물로 판단
        if min_distance < self.stop_distance:
            self.obstacle_detected = True
            self.last_obstacle_time = time.time()
        else:
            self.obstacle_detected = False

    def make_stop_cmd(self):
        # 모든 선속도/각속도를 0으로 만드는 정지 명령 생성
        stop = TwistStamped()
        stop.header.stamp = self.get_clock().now().to_msg()
        stop.header.frame_id = self.latest_cmd.header.frame_id

        stop.twist.linear.x = 0.0
        stop.twist.linear.y = 0.0
        stop.twist.linear.z = 0.0

        stop.twist.angular.x = 0.0
        stop.twist.angular.y = 0.0
        stop.twist.angular.z = 0.0

        return stop

    def timer_callback(self):
        now = time.time()

        # 1. 갑툭튀 장애물 감지 → 즉시 정지
        if self.obstacle_detected:
            self.state = 'SUDDEN_OBSTACLE_STOP'
            out_cmd = self.make_stop_cmd()

        # 2. 장애물 제거 직후 → 일정 시간 대기
        elif now - self.last_obstacle_time < self.resume_delay:
            self.state = 'WAIT_AFTER_SUDDEN_STOP'
            out_cmd = self.make_stop_cmd()

        # 3. 평상시 → Nav2 속도 명령 그대로 전달
        else:
            self.state = 'NORMAL'
            out_cmd = self.latest_cmd

        # 최종 속도 명령 publish
        out_cmd.header.stamp = self.get_clock().now().to_msg()
        self.cmd_pub.publish(out_cmd)

        # 현재 safety 상태 publish
        state_msg = String()
        state_msg.data = self.state
        self.state_pub.publish(state_msg)

def main(args=None):
    rclpy.init(args=args)

    node = SafetyCmdVelNode()

    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

## 3. 최종 Launch 파일

경로:

```
~/setros_ws/src/setros_safety/launch/setros_nav2_safety.launch.py
```

역할:

```
Nav2와 setros_safety_node를 한 번에 실행하는 최종 시연용 launch 파일
```

최종 코드:

```python
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import PathJoinSubstitution, EnvironmentVariable
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    # ===== TurtleBot3 Navigation2 launch 포함 =====
    nav2_launch = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            PathJoinSubstitution([
                FindPackageShare('turtlebot3_navigation2'),
                'launch',
                'navigation2.launch.py'
            ])
        ),
        launch_arguments={
            # SLAM으로 생성한 최종 지도 사용
            'map': PathJoinSubstitution([
                EnvironmentVariable('HOME'),
                'setros_map_new.yaml'
            ]),

            # 속도, 가속도, costmap 등이 튜닝된 Nav2 파라미터 파일
            'params_file': PathJoinSubstitution([
                EnvironmentVariable('HOME'),
                'setros_nav2_params.yaml'
            ]),

            # Nav2 lifecycle 자동 활성화
            'autostart': 'true',
        }.items()
    )

    # ===== 갑툭튀 장애물 Emergency Stop safety node =====
    safety_node = Node(
        package='setros_safety',
        executable='safety_cmd_vel_node',
        name='setros_safety_node',
        output='screen',
        parameters=[{
            # 정면 ±25도 감시
            'front_angle_deg': 25.0,

            # 28cm 이내 장애물 감지 시 정지
            'stop_distance': 0.28,

            # 장애물 제거 후 1초 대기 후 재출발
            'resume_delay': 1.0,
        }]
    )

    return LaunchDescription([
        nav2_launch,
        safety_node,
    ])
```

---

## 4. `setup.py`

경로:

```
~/setros_ws/src/setros_safety/setup.py
```

최종 코드:

```python
import os
from glob import glob
from setuptools import find_packages, setup

package_name = 'setros_safety'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),

    data_files=[
        # ROS 2 패키지 인덱스 등록
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),

        # package.xml 설치
        ('share/' + package_name, ['package.xml']),

        # launch 파일 설치
        (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
    ],

    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='an',
    maintainer_email='an@todo.todo',
    description='SetROS safety command velocity filter for TurtleBot3 Nav2',
    license='TODO',
    tests_require=['pytest'],

    entry_points={
        'console_scripts': [
            # ros2 run setros_safety safety_cmd_vel_node 로 실행 가능하게 등록
            'safety_cmd_vel_node = setros_safety.safety_cmd_vel_node:main',
        ],
    },
)
```

---

## 5. `package.xml`

경로:

```
~/setros_ws/src/setros_safety/package.xml
```

핵심 dependency:

```xml
<depend>rclpy</depend>
<depend>geometry_msgs</depend>
<depend>sensor_msgs</depend>
<depend>std_msgs</depend>
```

사용 목적:

```
rclpy         : Python ROS 2 node 작성
geometry_msgs: TwistStamped 속도 명령 사용
sensor_msgs  : LaserScan 사용
std_msgs     : safety_state 문자열 출력
```

---

## 6. `/scan` 각도 디버깅 코드

`safety_node`가 장애물을 인식하지 못할 때 사용한 일회성 테스트 코드.

```python
python3-<< 'EOF'
import rclpy, math
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import LaserScan

class ScanAngleFinder(Node):
    def __init__(self):
        super().__init__('scan_angle_finder')

        # /scan은 sensor-data QoS로 구독해야 안정적으로 수신됨
        self.sub = self.create_subscription(
            LaserScan,
            '/scan',
            self.cb,
            qos_profile_sensor_data
        )

    def norm(self, deg):
        # 0~360도 또는 -180~180도 형식을 -180~180도로 통일
        return ((deg + 180.0) % 360.0) - 180.0

    def cb(self, msg):
        vals = []

        for i, r in enumerate(msg.ranges):
            if math.isinf(r) or math.isnan(r):
                continue
            if r < msg.range_min or r > msg.range_max:
                continue

            raw_deg = math.degrees(msg.angle_min + i * msg.angle_increment)
            norm_deg = self.norm(raw_deg)
            vals.append((r, raw_deg, norm_deg))

        # 가장 가까운 scan point 10개 출력
        vals.sort(key=lambda x: x[0])

        print("=== closest 10 scan points ===")
        for r, raw, norm in vals[:10]:
            print(f"distance={r:.3f} m | raw_angle={raw:.1f} deg | normalized_angle={norm:.1f} deg")

        rclpy.shutdown()

rclpy.init()
node = ScanAngleFinder()
rclpy.spin(node)
EOF
```

확인 결과:

```
distance=0.292 m | raw_angle=358.0 deg | normalized_angle=-2.0 deg
distance=0.293 m | raw_angle=0.0 deg | normalized_angle=0.0 deg
distance=0.293 m | raw_angle=1.0 deg | normalized_angle=1.0 deg
```

해석:

```
로봇 정면 장애물이 0도 근처에서 정상 감지됨.
따라서 전방 기준은 0도가 맞고, safety_node 문제는 /scan QoS 문제였음.
```

---

## 7. 최종 코드 구조 요약

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
        ↓
모터 제어
```

```
LiDAR /scan
        ↓
setros_safety_node
        ↓
전방 ±25도, 0.28m 이내 장애물 판단
        ↓
정상: /cmd_vel_nav 그대로 전달
위험: /cmd_vel = 0으로 차단
```

## 8. 최종 구현 요약

```
1. safety_cmd_vel_node.py
   - /cmd_vel_nav 구독
   - /scan 구독
   - 갑툭튀 장애물 감지
   - 위험 시 /cmd_vel 0 출력
   - /safety_state 상태 출력

2. setros_nav2_safety.launch.py
   - Nav2 실행
   - 저장된 지도 setros_map_new.yaml 적용
   - 튜닝 파라미터 setros_nav2_params.yaml 적용
   - safety node 자동 실행

3. setup.py 수정
   - safety_cmd_vel_node 실행 등록
   - launch 파일 설치 등록
```



---

