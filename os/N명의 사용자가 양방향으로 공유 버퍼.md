# 어떤 상황?
- N명의 사용자가 메시지를 보내기도 하고 받기도 하는 상황이다.
- 한 사용자가 메시지를 보내면 다른 사용자들이 모두 받는다.
- 카카오톡 단톡방이라고 생각하면 된다.

## 두가지 방법
- 메시지 큐 하나를 이용하는 방법
  - 메시지 큐를 사용하면 mtype 값을 조절해서 메시지 수신을 조절할 수 있다.
  - 컨트롤 메시지를 큐에 한개만 넣어두고 사용하면 참여자의 수, 메시지 번호를 관리할 수 있다.
  - 메시지 큐는 기본적으로 blocking 작업이기 때문에, 동기화되어 동작한다.
- 공유 메모리와 세마포를 이용하는 방법
  - 공유 메모리는 임의의 크기의 버퍼를 사용할 수 있다.
  - 공유 메모리를 구조체로 정의해서 참여자 정보와, 메시지 번호, 버퍼 등을 관리할 수 있다.
  - 세마포는 구현 방법에 따라 필요한 갯수가 달라진다. 참여자가 N명인 경우, p-c와 r-w 문제를 참고하면 N+2개의 세마포로 구현이 가능하다.
 
## 메시지 큐
- ```C
  #include <fcntl.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <unistd.h>
  #include <stdio.h>
  #include <string.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <signal.h>
  #include <sys/wait.h>
  #include <sys/mman.h>
  #include <sys/ipc.h>
  #include <sys/msg.h>
  #include <sys/sem.h>
  #include <sys/shm.h>
  
  struct q_entry{
          long mtype;
          int snum;
          int cnum;
          int s_id;
          char msg[512];
  };
  
  struct q_entry cmessage(int mtype, int snum, int cnum);
  struct q_entry nmessage(int mtype, int s_id, char *str);
  
  void do_writer(int qid, int id){
          char temp[512];
          int i, cnum, index;
          struct q_entry msg1, msg2;
  
          while(1){
                  scanf("%s", temp);
                  // (i) message 보내기 전 준비
                  msgrcv(qid, &msg1, sizeof(msg1)-sizeof(long), 1, 0);
                  cnum = msg1.cnum;
                  index = msg1.snum;
  
                  msg2=nmessage(index, id, temp);
                  for (i=0; i<cnum; i++) {
                          // (a) message 보내기
                          msgsnd(qid, &msg2, sizeof(msg2)-sizeof(long), 0);
                          sleep(1);
                  }
                  msg1.snum++;
                  msgsnd(qid, &msg1, sizeof(msg1)-sizeof(long), 0);
  
                  if (cnum==1)
                      printf("id=%d, talkers=%d, msg#=%d ...\n", id, msg1.cnum, msg1.snum);
  
                  if (strcmp(temp, "talk_quit")==0)
                          break;
          }
  
          exit(0);
  }
  
  void do_reader(int qid, int id, int index){
          int cnum;
          struct q_entry msg;
  
          while(1){
                  // (j) message 받기
                  msgrcv(qid, &msg, sizeof(msg)-sizeof(long), index, 0);
                  if (msg.s_id!=id){
                          printf("[sender=%d & msg#=%ld] %s\n", msg.s_id, msg.mtype, msg.msg);
                  }
                  // (c) message 받은 후 필요한 작업
                  else if (strcmp(msg.msg, "talk_quit")==0)
                          break;
  
                  index++;
          }
  
          exit(0);
  }
  
  int main(int argc, char **argv) {
          int i, qid, id, index;
          pid_t pid[2];
          key_t key;
          struct q_entry msg1, msg2;
  
          key=ftok("key", 5);
          // (k) message queue 만들고 초기화 작업
          qid = msgget(key, 0600|IPC_CREAT|IPC_EXCL);
          if (qid == -1) {
                  qid = msgget(key, 0600);
          }
          else {
                  msg2 = cmessage(1, 2, 0);
                  msgsnd(qid, &msg2, sizeof(msg2)-sizeof(long), 0);
          }
  
          id=atoi(argv[1]);
          // (l) 통신 전 필요한 작업
          msgrcv(qid, &msg1, sizeof(msg1)-sizeof(long), 1, 0);
          msg1.cnum++;
          index = msg1.snum;
          msgsnd(qid, &msg1, sizeof(msg1)-sizeof(long), 0);
          printf("id=%d, talkers=%d, msg#=%d ...\n", id, msg1.cnum, msg1.snum);
  
          // (d) 함수 호출해서 message 주고 받기
          if (fork() == 0) {
                  do_writer(qid, id);
                  exit(1);
          }
          else if (fork() == 0) {
                  do_reader(qid, id, index);
                  exit(1);
          }
  
          wait(0);
          wait(0);
  
          // (h) message 통신 완료 후 message queue 지우기
          msgrcv(qid, &msg1, sizeof(msg1)-sizeof(long), 1, 0);
          if (msg1.cnum == 1)
                  msgctl(qid, IPC_RMID, 0);
          else {
                  msg1.cnum--;
                  msgsnd(qid, &msg1, sizeof(msg1)-sizeof(long), 0);
          }
  
          exit(0);
  }
  
  struct q_entry cmessage(int mtype, int snum, int cnum){
          struct q_entry msg;
  
          msg.mtype=mtype;
          msg.snum=snum;
          msg.cnum=cnum;
          msg.s_id=0;
          strcpy(msg.msg, "");
  
          return msg;
  }
  
  struct q_entry nmessage(int mtype, int s_id, char *str){
          struct q_entry msg;
  
          msg.mtype=mtype;
          msg.snum=0;
          msg.cnum=0;
          msg.s_id=s_id;
          strcpy(msg.msg, str);
  
          return msg;
  }
  ```

## 공유 메모리와 세마포
- ```C
  #include <stdio.h>
  #include <stdlib.h>
  #include <sys/types.h>
  #include <sys/ipc.h>
  #include <sys/sem.h> /* semaphore */
  #include <sys/shm.h> /* shared memory */
  #include <sys/wait.h>
  #include <unistd.h>
  #include <string.h>
  #include <time.h>
  
  #define R 4
  #define B 10
  
  #define SEM_EMPTY 0
  #define SEM_MUTEX 5
  
  struct databuf1{
      int s_id;
      char msg[512];
  };
  
  struct databuf2{
      int index;
      int members[R];
      struct databuf1 buf[B];
  };
  
  // union semun{
  //     int val;
  //     struct semid_ds *buf;
  //     ushort *array;
  // };
  
  void sem_wait(int semid, int semidx);
  void sem_signal(int semid, int semidx);
  
  void do_writer(int id, struct databuf2 *buf, int semid) {
      int i, index, cnt;
      char temp[512];
  
      while(1){
          scanf("%s", temp);
  
          // 버퍼에 쓸 공간이 있으면서, 상호배제를 획득한 뒤 쓰기 작업을 수행한다.
          sem_wait(semid, SEM_EMPTY);
          sem_wait(semid, SEM_MUTEX);
  
          // 쓰기 작업
          // 현재 써야 할 위치는 index % B 이다.
          // s_id는 내 id, msg는 사용자 입력 temp
          // 다음 쓰는 사람이 작성할 위치로 index를 1 증가시킨다.
          index = buf->index;
          buf->buf[index%B].s_id = id;
          strcpy(buf->buf[index%B].msg, temp);
          buf->index++;
  
          // 새로운 메시지를 작성했다고 모든 참여자에게 알린다.
          cnt = 0;
          for (i=0; i<R; i++) {
              if (buf->members[i]!=-1) {
                  sem_signal(semid, i+1);
                  cnt++;
              }
          }
          // 현재 참여자가 나 혼자라면 메시지 출력
          if (cnt == 1) {
              printf("id=%d, talkers=%d, msg#=%d ...\n", id, cnt, buf->index%B);
          }
          // 작업을 완료했으니 상호배제를 해제한다.
          sem_signal(semid, SEM_MUTEX);
  
          // 종료 조건
          if (strcmp(temp, "talk_quit")==0) {
              break;
          }
      }
  
      exit(0);
  }
  
  void do_reader(int id, int rindex, struct databuf2 *buf, int semid) {
      int i, index, sender, cnt;
      char temp[512];
  
      // 읽어야 할 초기 위치를 전달받는다.
      index = rindex;
      while (1){
          // 내 우편함에 새로운 메시지가 도착했다면 통과한다. (writer가 넣어줌)
          sem_wait(semid, id);
  
          if (id == 4) {
              sleep(5);
          }
  
          // 읽기 작업
          // 현재 읽어야 할 위치는 index % B 이다.
          // sender는 메시지를 보낸 사람의 id, temp는 메시지 내용
          sender = buf->buf[index%B].s_id;
          strcpy(temp, buf->buf[index%B].msg);
  
          // 다음 읽을 메시지로 index를 1 증가시킨다. 그런데 이 작업은 상호배제를 획득하고 수행한다.
          // 만약 내가 마지막으로 읽은 사람이라면 버퍼를 비워줘야한다.
          // 그런데 상호배제를 획득하지 않고 index를 조정하면, 동시에 읽은 사람들이 생기고, 마지막으로 읽은 사람을 특정할 수 없다.
          sem_wait(semid, SEM_MUTEX);
          // 나는 읽었다는 표시를한다. 다음 index로 넘어간다.
          buf->members[id-1]++;
          cnt = 0;
          // 현재 참여중 (!= -1)인 사람들 중에서, 현재 index를 읽어야 하는 사람의 수를 확인한다.
          for (i=0; i<R; i++) {
              if (buf->members[i]!=-1) {
                  if (buf->members[i] <= index)
                      cnt++;
              }
          }
          // 만약 내가 가장 마지막으로 읽은 사람이라면, 버퍼 하나를 비워준다.
          if (cnt == 0) sem_signal(semid, SEM_EMPTY);
          // 작업이 끝났으니 상호배제를 해제한다.
          sem_signal(semid, SEM_MUTEX);
  
          // 내가 보낸 메시지가 아니면 메시지를 출력한다.
          if (sender != id) {
              printf("[sender=%d msg#=%d] %s\n", sender, index%B, temp);
          }
          // 내가 종료 메시지를 보냈으면 종료한다.
          else if (strcmp(temp, "talk_quit")==0) {
              break;
          }
  
          index++;
      }
  
      exit(0);
  }
  
  int main(int argc, char** argv){
      int id, semid, shmid, i, j, flag, cnum, rindex;
      ushort init[6] = {10, 0, 0, 0, 0, 1}; /* semaphore 초기값 설정. 비어있는 버퍼, 각 참여자의 우편함, 상호배제 */
      key_t key1, key2;
      pid_t pid[2];
      union semun arg;
      struct databuf2 *buf;
      key1=ftok("key", 3);
      if ((semid=semget(key1, 6, 0600|IPC_CREAT|IPC_EXCL))>0){
          arg.array=init;
          semctl(semid, 0, SETALL, arg);
      }
      else{
          semid=semget(key1, 6, 0);
      }
  
      key2=ftok("key", 5);
      if ((shmid=shmget(key2, sizeof(struct databuf2), 0600|IPC_CREAT|IPC_EXCL))>0) {
          buf=(struct databuf2 *)shmat(shmid,0,0);
          for (i=0; i<4; i++){
              buf->members[i]=-1;
          }
      }
      else{
          shmid=shmget(key2, sizeof(struct databuf2), 0600);
          buf=(struct databuf2 *)shmat(shmid,0,0);
      }
  
      id=atoi(argv[1]);
      // semaphore 작업
      sem_wait(semid, 5);
      // 읽기, 쓰기 하기 전에 필요한 작업
      if (buf->members[id-1]==-1){
          buf->members[id-1] = buf->index;
      }
      else{
          id = -1;
      }
      cnum = 0;
      for (i=0; i<4; i++){
          if (buf->members[i]!=-1){
              cnum++;
          }
      }
      rindex = buf->index;
      // semaphore 작업
      sem_signal(semid, 5);
  
      if (id==-1)
          return 0;
      printf("id=%d, talkers=%d, msg#=%d...\n", id, cnum, rindex%B);
  
      if (fork()==0){
          do_reader(id, rindex, buf, semid);
          exit(1);
      }
      else if (fork()==0){
          do_writer(id, buf, semid);
          exit(1);
      }
  
      for (i=0; i<2; i++){
          wait(0);
      }
      // semaphore 작업
      sem_wait(semid, 5);
      // 종료할 때 필요한 작업, 마지막으로 종료하는 talker이면, flag=0으로 설정
      buf->members[id-1] = -1;
      flag = 0;
      for (i=0; i<4; i++){
          if (buf->members[i]!=-1){
              flag = 1;
              break;
          }
      }
      // semaphore 작업
      sem_signal(semid, 5);
      if (flag==0){
          semctl(semid, IPC_RMID, 0);
          shmctl(shmid, IPC_RMID, 0);
          printf("sem & shm IPC_RMID...\n");
      }
      exit(0);
  }
  
  void sem_wait(int semid, int semidx){
      struct sembuf p_buf={semidx, -1, 0};
      if (semop(semid, &p_buf, 1)==-1)
          printf("semwait fails...\n");
  }
  
  void sem_signal(int semid, int semidx){
      struct sembuf p_buf={semidx, 1, 0};
      if (semop(semid, &p_buf, 1)==-1)
          printf("semsignal fails...\n");
  }
  ```
