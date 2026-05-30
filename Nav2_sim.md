# Nav2 노드별 역할 및 구조

Nav2 

: 로봇이 목적지까지 장애물을 피해 스스로 찾아갈 수 있게 해주는 역할. 사람이 지도를 보고 운전하는 것과 같은 과정을 여러 노드들이 나누어 수행함

- 주요 노드
    - Map server
        
        로봇이 다닐 공간의 지도를 불러오고 관리
        
    - AMCL
        
        센서 데이터를 지도와 대조해 현재 위치 계산
        
    - Planner Server
        
        길잡이. 목적지까지 가는 가장 효율적인 큰 경로 설계
        
    - controller server
        
        장애물을 피하며 바퀴에 직접 속도 명령 내려 로봇 움직임
        
    - bt navigator
        
        행동 트리를 통해 상황별 의사결정 (정지, 회전) 등을 내림
        
    - Recoveries Server
        
        로봇이 길을 잃거나 구석에 끼었을 때 탈출 동작 수행
        
- 노드 소통 구조
    
     bt navigator 라는 총괄팀장이 planner(경로 설계 담당)와 controlller(운전)에게 업무를 주고, amcl과 map server가 제공하는 위치 정보를 공유하며 소통하는 구조. 이 노드들은 action과 TF 이라는 공동 규격으로 실시간 대화를 나누기에, 로봇이 끊김 없이 주행을 수행할 수 있다. 
    
    **TF? 좌표 통역사 역할
    
    - TF Tree  : nav2에서 가장 중요한 세 가지 기준점
        
        map : 변하지 않는 지도의 원점
        
        odom : 로봇 출발 후 바퀴가 굴러간 거리를 계산한 위치 (상대 좌표)
        
        base_link : 현재 로봇 그 자체의 중싱점
        
    
    TF tree의 세가지 위치가 각자의 언어로 위치를 말할 때, 모두가 알아듣게 공통 단위를 실시간 통역해주는 역할
    
    **action? 피드백이 있는 업무 보고 
    
        시간이 오래 걸리는 작업을 시킬 때 사용하는 통신 방식. ‘목적지까지 가줘~’ 라는 명령은 1초만에 끝나지 않고, 가는 도중 로봇이 잘 가고 있는지, 막혔는지 등을 계속 확인해야함. 액션은 3단계 보고 체계로 이루어져 있음
    
    goal (목표 잡기) → feedback(중간 보고) → result(최종 결과)
    
    음식 주문과 비슷하다
    
    주문 넣으면(골) 조리중이라 알려주고(피드백)  마지막에 음식이 나옴 (결과)
    
    Nav2안에서 TF와 action은 어떻게 협업할까?
    action을 통해 저기 목표로 가!!라고 명령을 내리면, 로봇은 이동하는 내내 TF를 확인하며 지도 map 좌표 기준으로 내 몸인 base_link가 어디에 있는지를 실시간으로 계산, 그 계산 결과를 action의 feednack 으로 보내주어 우리가 rviz 화면에서 로봇이 움직이는걸 볼 수 있게 함. 
    

 가제보 + nav2(Rviz) 실행시 $ros2 node list 
→ 주요 노드들이 잘 돌아가고 있는지 확인 가능. 

- 위치 정보
    
    로봇이 위치를 인지하기 위해서는 서로 다른 기준점들이 유기적으로 연결되어야 함. 
    
    - map
        
        지도의 고정된 원점
        
    - odom
        
         로봇이 출발 후 바퀴 회전 등을 통해 계산한 상대적 이동 거리 
        
    - base_link
        
        로봇의 물리적 중심점
        
    
    map → odom → base_link 순으로 연결 구조가 흘러가야 로봇이 길을 잃지 않음
    
- 작동 흐름
    
    초기화(lifecycle) → 위치추정(localization)→경로계획(global planning)→주행 회피 (local control) → 도착
    
    초기화 : 모든 노드가 준비 되었는지 확인 후 활성화
    
    위치 추정 : 로봇이 자신의 위치를 지도 위에서 확정
    
    경로 계획 : 목적지까지 밑그림 그림
    
    주행 및 회피 : 장애물 감지하며 실시간 이동
    
    Nav2 실행 후 초기 위치 설정 → 목표 위치 이동할 때 시각적ㅇ로 확인 가능함

  # Nav2 실행 - 명령어 위주 정리

우선 nav2_ws → colcon build

sudo apt update

sudo apt install ros-jazzy-navigation2

- 터틀봇 설치

sudo apt install ros-jazzy-turtlebot3*

echo 'export TURTLEBOT3_MODEL=burger' >> ~/.bashrc

source ~/.bashrc

ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

→ 가제보 띄워지는지 확인
가제보 연 상태에서 nav2 도 열어줘야함 

일단 다른 터미널에 $ros2 pkg list | grep nav 열어서, 리스트에

nav2_controller (컨트롤서버 본체, 실제 속도 cmd_vel 계산)
nav2_dwb_controller (장애물 피하면서 경로 따라감)
nav2_regulated_pure_pursuit_controller (dwb보다 단순, 부드러운 움직임)

이거 세개 있는지 확인

없으면 ,
sudo apt install ros-jazzy-nav2-dwb-controller
sudo apt install ros-jazzy-nav2-regulated-pure-pursuit-controller
sudo apt install nav2_controller 

이제 가제보 + nav2 열기 (터미널 2개 사용), 각각
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

→ 가제보 열림!
<img width="2880" height="1734" alt="%E1%84%8B%E1%85%B5%E1%84%86%E1%85%B5%E1%84%8C%E1%85%B5%202026 %205 %2013 %20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%209 16" src="https://github.com/user-attachments/assets/23d3bd0f-f53d-44a4-b708-3bf4792505a4" />

가제보 띄운 후 새 터미널 하나 열어서, 

$source /opt/ros/jazzy/setup.bash
$export TURTLEBOT3_MODEL=burger
$ros2 launch turtlebot3_navigation2 navigation2.launch.py use_sim_time:=True autostart:True
→ nav2+ Rviz 열림!
<img width="1440" height="867" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202026-05-13%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%209 31 27" src="https://github.com/user-attachments/assets/e1f43031-ee61-45de-bdeb-565aa35cbece" />

Rviz 열렸을 때,

navigation : active
localization :active 이렇게 나와야 함 (노드가 서로 통신하며 사라짐, 시간 좀 걸릴수도)

죽어있다면
$ros2 lifecycle set /planner_server activate
$ros2 lifecycle set /controller_server activate
$ros2 lifecycle set /bt_navigator activate

Start up 버튼의 역할 : 버튼 누르면 신호를 받은 각 노드들은 필요한 값을 불러오고 통신 준비 마친 뒤, 활성화 (active) 상태로 전환됨
$ros2 launch turtlebot3_navigation2 navigation2.launch.py use_sim_time:=True autostart:=True << 여기서 autostart:=True가 ‘스마트 자동 시동’ 의 역할을 해서, 매번 [startup]을 누르는 번거로움을 줄여줌!

이제 초기 위치 설정 → 목표 위치 설정 하면 로봇이 움직이는거 확인 가능

먼저 가제보 화면에서 로봇의 위치와 방향 확인 → 상단 [2D pose estimate] → 위치와 방향에 맞춰 rviz화면에 표시하기!! (클릭한 상태로 드래그하면 초록 화살표 나옴)
<img width="306" height="560" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202026-05-14%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%205 02 56" src="https://github.com/user-attachments/assets/cce0ef87-6558-456f-be7f-6f07ff004d8e" />

초록점들 (particle)은 ‘로봇이 이 어디쯤 있다’ 알리는 역할 (amcl이 생성한 분포임)
컬러 그림자(잔상)은 현재 시스템이 계산중인 로봇의 가상 위치로, 가제보 속 실제 로봇 위치와 이 그림자가 일치해야 내비게이션이 길을 잃지 않고 주행 가능함. 

로봇은 지도만으로 자신의 위치를 알 수 없음. 초기 위치 설정으로 ‘여기에 있다’ 표시를 해주면, 그걸 힌트 삼아 로봇은 초록 파티클을 뿌려본 후, LiDAR로 주변 벽 까지의 거리를 잼 ⇒ 파티클 중 실제 벽과 거리가 맞는 것들을 찾아 초록 점들을 모으고 그림자를 고정할 수 있음

목표 지점 설정은 상단 [nav2 goal] → 원하는 지점과 방향에 맞춰 화살표 찍기. 이때 방향은 로봇이 도착했을떄 바라보아야 하는 방향으로 지정. 

지도의 검은색 /회색 / 흰색이 각각 갖는 의미가 있음

- 검은색 : 장애물이 있어 로봇이 ‘통행 금지 구역’으로 인식
- 회색 : 가보지 않아 정보 없는 구역
- 흰색 : 장애물의 없는 깨끗한 바닥이라 인식

따라서, 목표 (도착) 지점은 흰 바닥으로 선택해보는 것이 안전하다.
<img width="1440" height="867" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202026-05-14%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%203 52 57" src="https://github.com/user-attachments/assets/ab13822e-5e63-4723-97bf-5f8cc7d5f7f5" />

[nav2 goal] 화살표를 지정하면, 목적지를 나타내는 긴 선 하나가 나타나고, 이는 전역 경로를 보여줌. 사진과 같이 로봇이 움직이기 시작하며 나타나는 빨간선은 local path로, 로봇이 움직일때 마다 실시간으로 변하며 로봇의 ‘실전 운전 로직’을 보여줌.   

전체 경로가 네비게이션의 안내라면, 빨간 선은 운전자 판단에 의한 실제 핸들 조작

빨간 선이 부드럽다면 제어가 안정적인 상태, 요동친다면 속도 설정이 높아 로봇이 갈팡질팡하고 있다는 뜻이기도 하다.
<img width="1440" height="867" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202026-05-14%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%203 53 19" src="https://github.com/user-attachments/assets/b45acea7-4fa0-431f-a19e-888ddc4fd16a" />

Rviz 화면 상에서도 로봇이 움직이고, 가제보 화면을 확인해보면 여기서도 지정한 목표 지점을 향해 터틀봇이 움직이는 걸 확인할 수 있다. 이후 목적지에 도달하면 rviz 화면에 과정이 끝났다는 문구가 나타난다
