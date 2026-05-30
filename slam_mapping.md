우선 https://emanual.robotis.com/docs/en/platform/turtlebot3/quick-start/#pc-setup 요거 전부 설치완료

(

sudo apt update

sudo apt install ros-jazzy-turtlebot3-gazebo \
ros-jazzy-turtlebot3-simulations \
ros-jazzy-slam-toolbox \
ros-jazzy-navigation2 \
ros-jazzy-nav2-bringup

확인 필요

)

Turtlebot 모델지정

export TURTLEBOT3_MODEL=burger

gazebo 시뮬 실행

ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

Slam Toolbox를 비동기 모드로 실행

ros2 launch slam_toolbox online_async_launch.py use_sim_time:=True

지도 확인용 rviz2 실행

ros2 run rviz2 rviz2

rviz설정
1. Fixed Frame을 map으로 변경.
2. Add 버튼 -> By topic 탭 -> /map, /scan 추가
3. /tf 도 추가
   
!image.png

Teleop으로 로봇 조종 (맵핑 시작) 
ros2 run turtlebot3_teleop teleop_keyboard

에러발생시 입력

export TURTLEBOT3_MODEL=burger

