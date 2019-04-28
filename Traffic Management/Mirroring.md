## Mirroring
"shadowing"이라고도 불리는 Traffic mirroring은 아주 적은 risk로 어떤 변경사항을 production으로 배포가능하게 해준다. 
Mirroring은 live traffic을 복사해서 mirroed serivce로 똑같이 보내준다. 이런 mirrored traffic은 주요한 service의 critical request path에 대한 처리영역 밖(out of band)에서 일어나게 된다. 
