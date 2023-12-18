# FIFO
- named pipe 라고 한다. 파일 시스템 내에 특별한 파일로 존재해서 부모 자식을 제외한 프로세스간에도 데이터를 전달할 수 있다.
- pipe()는 익명 파이프라서 생성 시 읽기 쓰기 fd를 모두 전달하지만, FIFO는 open()으로 필요한 권한으로 열면 된다.
```c
int main() {
  int fd, n_fd;

  mkfifo("fifo", 0666);
  mkfifo("nb_fifo", 0666);

  fd = open("fifo", O_WRONLY);
  n_fd = open("nb_fifo", O_WRONLY | O_NONBLOCK);

  write(fd, "Hello, FIFO!\n", 13);

  exit(0);
}  
```
- 보통은 O_RDONLY, O_WRONLY 둘 중 하나로 오픈하는데, O_RDWR 으로 오픈하는 경우가 몇가지 있다.
  1. 단일 프로세스 내의 읽기/쓰기
    - 테스트하거나 특별한 구현에 사용될 수 있다. 특별한 구현은 복잡한 설계가 필요하다.
  2. 블록킹 방지
    - FIFO는 O_RDONLY로 오픈될 경우, O_WRONLY가 열릴 때 까지 블록킹된다.
    - 그래서 O_RDWR으로 오픈하면 blocking 없이 진행할 수 있다.
  3. Reader의 입장에서 writer가 없는 상황 대비
    - O_RDONLY로 열었으면 writer가 없을 때 무한히 0을 리턴한다.
    - O_RDWR으로 열었으면 writer가 없을 때 blocking 상태로 대기한다.
