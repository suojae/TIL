
```plantuml
@startuml
' 시퀀스 다이어그램 전반 스타일(선택사항)
skinparam note {
  BackgroundColor #FFEEEE
  BorderColor #FF5555
  FontColor #FF0000
}

actor 사용자 as User
participant "QuestionListScreenState" as QLSS
participant Geolocator as GEO
participant "QuestionListViewModel" as VM
participant "TranslatableGoogleMapState" as TGM
participant Timer as TMR

'-----------------------------
' 앱 실행 & 초기화
'-----------------------------
User -> QLSS: 앱 실행(생성자 호출)
activate QLSS
QLSS -> QLSS: initState() 실행
note over QLSS
  - _initializeLifecycleListener()
  - _initializeLocationTracking()
  - _initializePagingController()
  - _panelController.addListener(...)
end note

'-----------------------------
' 위치 권한 요청 & 초기 위치
'-----------------------------
QLSS -> GEO: requestPermission()
GEO -> QLSS: 권한 승인(onGranted 콜백)
QLSS -> GEO: getCurrentPosition()
note right
  위치정보 사용 가능 상태
end note
GEO --> QLSS: Position(현재위치)
QLSS -> QLSS: _handleInitialPosition() 호출
note over QLSS
  - _isInitialLocationLoaded = true
  - _locationTimeoutTimer.cancel()
end note
QLSS -> VM: updateCurrentPosition(latitude, longitude)
QLSS -> TGM: setPosition(LatLng, panelPosition)
deactivate GEO

'-----------------------------
' 위치 타임아웃 처리
'-----------------------------
QLSS -> TMR: _locationTimeoutTimer 시작
TMR -> QLSS: 타이머 만료(위치 못 받음)
note over QLSS #FFEEEE
  **문제 지점**  
  - 실제 위치를 받기 전에 타이머가 만료되면,
    _isInitialLocationLoaded가 false로 간주되어
    '기본위치'로 초기화
  - 이후 늦게 도착한 실제 위치는
    getQuestions() 재호출 없이
    지도만 이동될 가능성 있음
end note
QLSS -> QLSS: if !_isInitialLocationLoaded
note over QLSS
  - 기본위치 사용
  - getQuestions(DEFAULT_LOCATION)
  - setPosition(DEFAULT_LOCATION)
end note

'-----------------------------
' 지도 초기화 & Panel 설정
'-----------------------------
QLSS -> QLSS: _initializePanel()
QLSS -> TMR: _panelInitTimer 시작
TMR -> QLSS: _panelInitTimer 만료
QLSS -> QLSS: _panelController.moveToLevel(2)
note over QLSS
  - 패널 위치 이동 후 onPanelSlide 콜백
end note
QLSS -> TGM: translateCamera(mapOffset)
QLSS -> TGM: setPosition(현재위치 or 기본위치, panelPosition)

'-----------------------------
' 질문 데이터 로딩
'-----------------------------
QLSS -> VM: getQuestions(latitude, longitude, page=0)
activate VM
VM -> VM: 서버/레포지토리에서 Question 목록 조회
VM -> QLSS: 질문 데이터 반환
deactivate VM
QLSS -> QLSS: _pagingController에 질문 데이터 추가

'-----------------------------
' 위치 스트리밍
'-----------------------------
GEO -> QLSS: getPositionStream()으로 Position 이벤트 발생
QLSS -> QLSS: if position != null
QLSS -> VM: updateCurrentPosition()
QLSS -> TGM: setPosition(새로운 위치, panelPosition)

@enduml
```

1. **현재 위치를 받으면(onGranted 콜백 성공 시점):**
- 이미 `_locationTimeoutTimer`가 만료되었더라도, 
- **강제로 다시** `getQuestions(실제위치)`를 호해 **질문 목록**과 **지도 위치**가 동기화되도록 합니다.


2. **혹은 타임아웃 방식을 없애거나 더 길게 주고,**
- 위치가 늦게 오더라도 `DEFAULT_LOCATION`으로 질문을 먼저 보여주고,
- **뒤늦게라도** 위치가 오면 `getQuestions(실제위치)`를 다시 호출하여 데이터를 새로 불러옵니다.


3. **만약 최대한 빨리 사용성(지도 표시, 질문 표시)을 보장해야 한다면,**
- 초기에만 기본 위치로 빠르게 세팅 후,
- 위치가 도착하면 **바로** 현재 위치로 재세팅 + `getQuestions()` 재호출  
- 이런 식으로 **두 번 로드**하는 전략을 사용합니다(UX 트레이드오프 고려).