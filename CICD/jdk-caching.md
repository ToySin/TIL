# Github Action Cache
- 안썼다! 왜냐면.. 뭐가 잘 안됐다 ㅋㅋ

# Set up JDK
- 여기에서 간단하게 제공하는 cache 기능이 있다.
- 그래서 그냥 gradle 캐싱하라고 한 줄만 추가했다.

# 그랬더니
- 빌드에 6분 걸리던게 (JDK 설치 3분 + gradle 빌드 2분 + 잡동사니 1분)
- 이제 1분 걸린다. (JDK 캐싱 1초 + gradle 빌드 10초 + 잡동사니 1분)
