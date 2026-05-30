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
   
<img width="677" height="273" alt="image" src="https://github.com/user-attachments/assets/c62e7b90-938e-45fc-8754-d37e6f81138b" />


Teleop으로 로봇 조종 (맵핑 시작) 
ros2 run turtlebot3_teleop teleop_keyboard

에러발생시 입력
export TURTLEBOT3_MODEL=burger
<img width="1455" height="922" alt="image-2" src="https://github.com/user-attachments/assets/18719330-542e-4ca3-abcf-fb49ceb6ae61" />

cmd창 누르고 이동
<img width="455" height="467" alt="image-3" src="https://github.com/user-attachments/assets/394555ab-fb59-4eb9-b048-d5fa920dc2c4" />
지도 제작됨 

그후 저장후 끝





export TURTLEBOT3_MODEL=
