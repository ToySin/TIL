# 단일 레포지토리 배포
- 도커를 사용해서 이미지를 빌드하고 컨테이너를 생성하는 편리함을 배웠다.
- 우선은 스프링부트를 도커를 이용해서 EC2에 배포하는 과정을 복습해보자.

# 도커 파일
- 디렉토리 구조는 다음과 같다.
  ```
  Project/
  ㄴ build/
  ㄴ gradle/
  ㄴ src/
    ㄴ main/
    ㄴ test/
  ㄴ Dockerfile
  ㄴ gradlew
  ...
  ```
- Dockerfile은 다음과 같다.
  ```
  FROM openjdk:21
  ARG JAR_PATH=./build/libs
  
  WORKDIR /usr/src/app
  
  COPY ${JAR_PATH}/agreement-of-daily-use-0.0.1-SNAPSHOT.jar app.jar
  EXPOSE 8080/tcp
  
  ENTRYPOINT ["java","-jar","app.jar"]
  ```
- 나중에 추가될 수 있겠지만, 현재는 단순하게 작성했다.
  1. openjdk:21 버전 이미지를 기반으로 이미지를 생성한다.
  2. 이미지 청사진의 작업 디렉토리를 /usr/src/app으로 변경한다. 디렉토리가 없으면 만들고 이동한다.
  3. docker bulid가 수행되는 경로(나중에 트러블 슈팅이 있다)의 ./bulid/libs 위치에 있는 jar 파일을 이미지 청사진에(위치: /usr/src/app) 복사해서 넣는다.
  4. 8080포트로 tcp 연결을 받도록 설정한다.
  5. 컨테이너가 실행될 땐 `java -jar app.jar` 명령을 실행한다.
- 환경변수가 추가되면 나중에 다시 작성하겠다. 

# 도커 이미지 생성 및 업로드
- 워크플로 파일은 다음과 같다.
  ```
  name: Docker Image CI

  on:
    push:
      branches: [ "main" ]
  
  env:
    DOCKER_IMAGE: ghcr.io/toysin/agreement-service-auto-deploy
    VERSION: ${{ github.sha }}
    DOCKER_CONTAINER: agreement-app
  
  jobs:
    # 빌드 Job
    build:
      name: Image Build and Push
      runs-on: ubuntu-latest
  
      steps:
      - uses: actions/checkout@v3
  
      # Build를 위한 Java 환경 설정
      - name: Set up JDK 21
        uses: actions/setup-java@v4.0.0
        with:
          java-version: 21
          distribution: oracle
  
      # Gradle으로 프로젝트 빌드 및 .jar 파일 생성
      - name: Gradle Build Action
        uses: gradle/gradle-build-action@v3.0.0-beta.3
        with:
          arguments: build
          
      # Docker Image Build 수행
      - name: Build Docker Image
        run: docker build . -t ${{ env.DOCKER_IMAGE }}:latest
  
      # Docker Login 수행 (ghcr.io)
      - name: Login Docker
        uses: docker/login-action@v3.0.0
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.CR_PAT }}
      
      # Docker Image Push 수행
      - name: Push Docker Image
        run: docker push ${{ env.DOCKER_IMAGE }}:latest
      ...
  ```
- 어려웠던 점은, 도커 이미지를 빌드할때 도커에서 제공하는 액션을 사용해서(uses) 작업하는 경우에 파일 디렉토리 위치문제가 발생했다.
- docker build가 수행되는 경우에서 ./build/libs 위치를 참조해야하는데, 로그를 살펴보면 /tmp/mount15234.../build/libs의 jar 파일을 찾고 있었다.
- 실제로 참조해야하는 위치는 /home/runner/repo-name/repo-name/build/libs였다.
- 그래서 도커 액션을 사용하지않고, 직접 명령어를 실행해서(run) 이미지를 빌드했다.
- 여기까지 진행하면 github action의 runner에 docker image가 하나 생성된 상태이다.
- 도커 로그인을 수행하고 레포지토리에 이미지를 업로드한다.

# ec2에 배포하기
- 워크플로에서 배포 부분은 다음과 같다. 위 파일의 아래에 이어지는 내용이다.
  ```
    ...
    deploy:
    needs: build
    name: Deploy
    runs-on: [ self-hosted, label-sfc ]
    steps:
      # Docker Login 수행 (ghcr.io)
      - name: Login Docker
        uses: docker/login-action@v3.0.0
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.CR_PAT }}
      # Docker Container 생성
      - name: Create Docker Container
        run: |
          docker stop ${{ env.DOCKER_CONTAINER }} && docker rm ${{ env.DOCKER_CONTAINER }} && docker rmi ${{ env.DOCKER_IMAGE }}:latest
          docker pull ${{ env.DOCKER_IMAGE }}:latest
          docker run -d -p 8888:8080 --name ${{ env.DOCKER_CONTAINER }} --restart always ${{ env.DOCKER_IMAGE }}:latest
  ```
- 우선은 우리는 아직 github action의 runner에서 작업하고 있다.
- 하지만 ec2에서 배포를 수행해야한다.
- github action 탭에서 살펴보면 다른 실행환경에 runner를 파견보낼 수 있는 방법이 나와있다.
- 그 방법으로 ec2에 runner를 하나 파견한다. 이때 runner를 식별할 수 있는건 [ self-hosted, label-sfc ]이다.
- self-hosted는 필수고, label-sfc가 파견나간 러너의 식별자라고 보면 된다.
- 여튼 그녀석이 파견나가서 실행할 내용을 작성해준 내용이다.
- 도커 로그인을 수행하고 현재 실행중인 컨테이너와 이미지를 삭제, 새로운 이미지를 받아와서 실행한다.
- 테스트 서버로 8888포트로 연결해서 열어줬다.
- 나중에 프로필 별 설정은 공부하고 적용해보자.
